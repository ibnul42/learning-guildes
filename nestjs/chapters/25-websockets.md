# Chapter 25 — WebSockets and Gateways

## Learning Objectives

- Build a gateway with lifecycle hooks and typed message handlers
- Authenticate a socket at the handshake rather than per message
- Use rooms to target broadcasts without leaking across tenants
- Scale gateways across replicas with the Redis adapter
- Apply guards, pipes, and filters in a context that has no HTTP request

---

## 25.1 A Gateway Is a Controller for Sockets

```bash
npm i @nestjs/websockets @nestjs/platform-socket.io socket.io
```

```ts
// src/chat/chat.gateway.ts
@WebSocketGateway({
  namespace: '/chat',
  cors: { origin: process.env.CORS_ORIGINS?.split(',') ?? false, credentials: true },
})
export class ChatGateway {
  @WebSocketServer()
  private readonly server!: Server;

  @SubscribeMessage('message:send')
  handleMessage(
    @MessageBody() body: SendMessageDto,
    @ConnectedSocket() client: Socket,
  ): void {
    this.server.to(body.roomId).emit('message:new', {
      text: body.text,
      userId: client.data.user.id,
    });
  }
}
```

A gateway maps message names to methods the way a controller maps routes. The return value is sent back to the *sender* as an acknowledgement; broadcasting to others requires `this.server`.

> ⚠️ **Gotcha** — CORS on a gateway is configured separately from `app.enableCors()`. The HTTP setting does not apply to the WebSocket handshake, so an API with correct CORS can still have a gateway accepting connections from every origin — and nothing in the HTTP configuration hints at it.

Four lifecycle hooks cover the connection:

```ts
export class ChatGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  afterInit(server: Server) {}            // once, at startup
  handleConnection(client: Socket) {}     // per connection
  handleDisconnect(client: Socket) {}     // per disconnect — always clean up here
}
```

**Socket.IO versus native `ws`** is a real choice. Socket.IO adds rooms, automatic reconnection, acknowledgements, and a fallback transport, at the cost of a non-standard protocol requiring its client library. Native `ws` is a plain WebSocket any client can speak, and you build rooms and reconnection yourself. Choose Socket.IO unless a third party must connect with a standard client.

---

## 25.2 Authenticate at the Handshake

Authenticating per message is wasteful and leaves a window where an unauthenticated socket is connected. Do it once, at connection:

```ts
async handleConnection(client: Socket): Promise<void> {
  try {
    const token = this.extractToken(client);
    if (!token) throw new Error('No token');

    const payload = await this.jwt.verifyAsync<JwtPayload>(token);
    const user = await this.users.findActiveById(payload.sub);
    if (!user) throw new Error('Unknown user');

    // Attach once; every handler reads it without re-verifying.
    client.data.user = user;

    await client.join(`tenant:${user.tenantId}`);   // tenant isolation, immediately
  } catch {
    // Disconnect rather than leaving an unauthenticated socket connected.
    client.emit('error', { code: 'UNAUTHORIZED' });
    client.disconnect(true);
  }
}

private extractToken(client: Socket): string | undefined {
  return (
    client.handshake.auth?.token ??                       // preferred
    (client.handshake.headers.authorization as string)?.replace(/^Bearer /, '')
  );
}
```

**Do not put the token in the query string.** `?token=...` is logged by proxies, load balancers, and access logs, which turns every log store into a credential store. Socket.IO's `auth` payload is sent in the handshake body.

> ⚠️ **Gotcha** — A JWT verified at connection is never re-checked. A long-lived socket outlives the token's expiry, so a user who is banned or logs out keeps their connection indefinitely. Either re-validate periodically, or subscribe to a revocation channel and disconnect affected sockets (→ [Ch 21](21-authentication.md)).

---

## 25.3 Rooms

Rooms are how you target a broadcast. Every socket is automatically in a room named by its own id, and joins any others you specify:

```ts
await client.join(`task:${taskId}`);
this.server.to(`task:${taskId}`).emit('task:updated', payload);

// Everyone in the room except the sender
client.to(`task:${taskId}`).emit('task:updated', payload);
```

The distinction between `this.server.to(...)` and `client.to(...)` is exactly whether the sender receives their own event, and getting it wrong produces duplicated messages in the sender's UI — a bug that looks like a frontend problem.

**Room names are an authorization boundary.** Joining a room must be checked, or a client can subscribe to anything:

```ts
// ❌ The client picks the room; it can join any tenant's stream
@SubscribeMessage('room:join')
join(@MessageBody() room: string, @ConnectedSocket() client: Socket) {
  client.join(room);
}

// ✅ Verify access, and namespace the room so ids cannot collide
@SubscribeMessage('task:subscribe')
async subscribe(@MessageBody() dto: SubscribeDto, @ConnectedSocket() client: Socket) {
  const user = client.data.user;
  const task = await this.tasks.findAccessible(dto.taskId, user);
  if (!task) throw new WsException('Not found');
  await client.join(`task:${task.id}`);
}
```

---

## 25.4 Scaling Across Replicas

This is where most WebSocket deployments break. `this.server.emit()` reaches only sockets connected to **this process**. With three replicas behind a load balancer, a broadcast reaches roughly a third of your users.

```bash
npm i @socket.io/redis-adapter redis
```

```ts
// src/main.ts
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);

app.useWebSocketAdapter(new RedisIoAdapter(app, pubClient, subClient));
```

```ts
// src/adapters/redis-io.adapter.ts
export class RedisIoAdapter extends IoAdapter {
  constructor(
    app: INestApplication,
    private readonly pub: RedisClientType,
    private readonly sub: RedisClientType,
  ) { super(app); }

  createIOServer(port: number, options?: ServerOptions): Server {
    const server = super.createIOServer(port, options) as Server;
    server.adapter(createAdapter(this.pub, this.sub));   // broadcasts cross instances
    return server;
  }
}
```

Two other things break at more than one replica:

**Sticky sessions.** Socket.IO's HTTP long-polling fallback makes several requests during the handshake, and they must reach the same instance. Either enable sticky sessions on the load balancer or force `transports: ['websocket']` to skip polling entirely.

**In-memory connection state.** A `Map<userId, Socket>` on one instance knows nothing about the others. Any "is this user online" or "send to this user" feature needs shared state.

---

## 25.5 Guards, Pipes, and Errors

The pipeline works, with the transport caveat from [Chapter 12](12-guards.md):

```ts
@Injectable()
export class WsAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const client = context.switchToWs().getClient<Socket>();
    if (!client.data?.user) throw new WsException('Unauthorized');
    return true;
  }
}
```

Validation needs an explicit pipe, because there is no global HTTP pipe on this path:

```ts
@UsePipes(new ValidationPipe({ whitelist: true, transform: true }))
@SubscribeMessage('message:send')
handleMessage(@MessageBody() dto: SendMessageDto) {}
```

Errors use `WsException` and a `BaseWsExceptionFilter`, and there is no status code — an unhandled error in a handler **disconnects nothing and reports nothing** unless you handle it:

```ts
@Catch()
export class WsExceptionFilter extends BaseWsExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const client = host.switchToWs().getClient<Socket>();
    const message = exception instanceof WsException
      ? exception.getError()
      : 'Internal error';                     // never leak the real message
    client.emit('exception', { message });
  }
}
```

---

## 25.6 Common Mistakes and Best Practices

**Mistake: assuming HTTP CORS covers the gateway.** They are configured separately.

**Mistake: the token in a query string.** It ends up in every proxy and access log.

**Mistake: never re-validating an authenticated socket.** Connections outlive token expiry, logout, and bans.

**Mistake: letting clients choose room names.** It is a subscription to anything they can guess.

**Mistake: no Redis adapter with multiple replicas.** Broadcasts reach only the sockets on one instance.

**Mistake: in-memory presence maps.** They are per-process and silently wrong at more than one replica.

**Mistake: no cleanup in `handleDisconnect`.** Timers, subscriptions, and presence entries accumulate per connection.

**Best practice: authenticate once at the handshake and attach the principal to `client.data`.** Handlers then read it without re-verifying.

**Best practice: namespace room names by resource type.** `task:42` cannot collide with `user:42`.

**Best practice: force `transports: ['websocket']` behind a load balancer.** It removes the sticky-session requirement entirely.

---

## 25.7 Real-World Example: A Multi-Tenant Gateway That Scales

A gateway with handshake authentication, tenant-isolated rooms, authorized subscriptions, cross-replica broadcasting, and cleanup that actually runs.

```ts
// src/realtime/realtime.gateway.ts
@WebSocketGateway({
  namespace: '/realtime',
  cors: { origin: process.env.CORS_ORIGINS?.split(',') ?? false, credentials: true },
  transports: ['websocket'],          // no polling → no sticky sessions needed
  pingInterval: 25_000,
  pingTimeout: 20_000,
})
@UseFilters(WsExceptionFilter)
export class RealtimeGateway implements OnGatewayConnection, OnGatewayDisconnect {
  private readonly logger = new Logger(RealtimeGateway.name);

  @WebSocketServer()
  private readonly server!: Server;

  constructor(
    private readonly jwt: JwtService,
    private readonly users: UsersService,
    private readonly tasks: TasksService,
    private readonly presence: PresenceService,   // Redis-backed, not in-memory
  ) {}

  async handleConnection(client: Socket): Promise<void> {
    try {
      const token = client.handshake.auth?.token as string | undefined;
      if (!token) throw new WsException('Missing token');

      const payload = await this.jwt.verifyAsync<JwtPayload>(token);
      const user = await this.users.findActiveById(payload.sub);
      if (!user) throw new WsException('Unknown user');

      client.data.user = user;
      // Store the token's expiry so a long-lived socket can be closed when
      // its credential lapses, rather than living on indefinitely.
      client.data.expiresAt = payload.exp * 1000;

      await client.join(`tenant:${user.tenantId}`);
      await this.presence.markOnline(user.id, client.id);

      this.logger.debug(`${user.id} connected (${client.id})`);
    } catch (err) {
      client.emit('exception', { message: 'Unauthorized' });
      client.disconnect(true);          // never leave an unauthenticated socket open
    }
  }

  async handleDisconnect(client: Socket): Promise<void> {
    const user = client.data?.user as User | undefined;
    if (!user) return;
    // Cleanup must be unconditional: without it, presence entries accumulate
    // for every dropped connection and "online" becomes permanently wrong.
    await this.presence.markOffline(user.id, client.id);
  }

  @UseGuards(WsAuthGuard)
  @UsePipes(new ValidationPipe({ whitelist: true, transform: true }))
  @SubscribeMessage('task:subscribe')
  async subscribeToTask(
    @MessageBody() dto: SubscribeTaskDto,
    @ConnectedSocket() client: Socket,
  ): Promise<{ ok: true }> {
    const user = client.data.user as User;

    // Authorization on join. Without this the client picks its own room name
    // and can subscribe to any task in any tenant by guessing an id.
    const task = await this.tasks.findAccessible(dto.taskId, user);
    if (!task) throw new WsException('Task not found');

    await client.join(`task:${task.id}`);
    return { ok: true };                // acknowledgement to the sender only
  }

  @UseGuards(WsAuthGuard)
  @SubscribeMessage('task:unsubscribe')
  async unsubscribe(
    @MessageBody() dto: SubscribeTaskDto,
    @ConnectedSocket() client: Socket,
  ): Promise<{ ok: true }> {
    await client.leave(`task:${dto.taskId}`);
    return { ok: true };
  }

  /**
   * Called by TasksService after a write. Because the Redis adapter is
   * installed, this reaches subscribers on every replica, not just this one.
   */
  broadcastTaskUpdate(task: Task): void {
    this.server.to(`task:${task.id}`).emit('task:updated', {
      id: task.id,
      title: task.title,
      status: task.status,
      updatedAt: task.updatedAt,
    });
  }

  /** Closes sockets whose credential has expired. Scheduled every minute. */
  @Interval(60_000)
  async expireStaleSockets(): Promise<void> {
    const now = Date.now();
    const sockets = await this.server.fetchSockets();
    for (const socket of sockets) {
      if (typeof socket.data.expiresAt === 'number' && socket.data.expiresAt < now) {
        socket.emit('exception', { message: 'Token expired' });
        socket.disconnect(true);
      }
    }
  }
}
```

```ts
// src/realtime/presence.service.ts
@Injectable()
export class PresenceService {
  constructor(@Inject(REDIS_CLIENT) private readonly redis: RedisClientType) {}

  /**
   * A set of socket ids per user, in Redis rather than in a process-local Map.
   * One user can have several connections (tabs, devices), so "offline" means
   * the set is empty — not that one socket closed.
   */
  async markOnline(userId: string, socketId: string): Promise<void> {
    await this.redis.sAdd(`presence:${userId}`, socketId);
    await this.redis.expire(`presence:${userId}`, 3600);   // reaped if a crash skips cleanup
  }

  async markOffline(userId: string, socketId: string): Promise<void> {
    await this.redis.sRem(`presence:${userId}`, socketId);
  }

  async isOnline(userId: string): Promise<boolean> {
    return (await this.redis.sCard(`presence:${userId}`)) > 0;
  }
}
```

The decision worth defending is tracking presence as a *set* of socket ids in Redis rather than a boolean. The boolean is simpler and it is wrong in the ordinary case: a user with the app open in two tabs produces two connections, and closing one would mark them offline while they are still connected. A per-process `Map` fails differently and worse — it is correct on one instance and silently wrong across three, so presence works in development and degrades in production in a way no test catches. The `expire` on the key is the part that is easy to omit: `handleDisconnect` does not run if the process is killed, so without a TTL a crash leaves users permanently "online" with no mechanism to correct it. Any state keyed by connection needs a reaping strategy, because the cleanup path is exactly the one that does not run when things go wrong.

---

## 25.8 Questions

**1.** MCQ — A gateway's CORS is configured: a) by `app.enableCors()` b) in the `@WebSocketGateway()` options c) by Helmet d) automatically
<details><summary>Answer</summary>**b)** HTTP CORS does not apply to the WebSocket handshake.</details>

**2.** MCQ — `this.server.to(room).emit()` versus `client.to(room).emit()` differs in: a) speed b) whether the sender receives it c) reliability d) ordering
<details><summary>Answer</summary>**b)** `client.to()` excludes the sender.</details>

**3.** MCQ — Without the Redis adapter and three replicas, a broadcast reaches: a) everyone b) one third c) nobody d) the sender only
<details><summary>Answer</summary>**b)** Only sockets connected to the emitting process.</details>

**4.** MCQ — A token should be sent in: a) the query string b) the handshake `auth` payload c) a cookie only d) the first message
<details><summary>Answer</summary>**b)** Query strings are logged by proxies and load balancers.</details>

**5.** MCQ — In a WebSocket guard, the client comes from: a) `switchToHttp()` b) `switchToWs().getClient()` c) `@Req()` d) `getArgs()`
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Forcing `transports: ['websocket']` removes the need for: a) CORS b) authentication c) sticky sessions d) the Redis adapter
<details><summary>Answer</summary>**c)** Polling is what requires several handshake requests to hit the same instance.</details>

**7.** Short — Why authenticate at the handshake rather than per message?
<details><summary>Answer</summary>It verifies once instead of on every message, and it avoids a window in which an unauthenticated socket is connected and can send anything.</details>

**8.** Short — Why is a socket authenticated once a long-term problem?
<details><summary>Answer</summary>The connection outlives the token, so logout, a ban, or expiry does not disconnect it — you need periodic re-validation or a revocation channel.</details>

**9.** Short — Why must joining a room be authorized?
<details><summary>Answer</summary>Otherwise the client chooses its own room name and can subscribe to any resource whose id it can guess.</details>

**10.** Short — Why is presence a set rather than a boolean?
<details><summary>Answer</summary>One user commonly has several connections; a boolean marks them offline when any one closes.</details>

**11.** Short — Why does presence state need a TTL?
<details><summary>Answer</summary>`handleDisconnect` does not run if the process is killed, so without expiry a crash leaves entries that nothing will ever clean up.</details>

**12.** Output — A user opens two tabs, then closes one. With a boolean presence flag, what does `isOnline` return?
<details><summary>Answer</summary>`false`, incorrectly — the remaining tab is still connected.</details>

**13.** Output — `client.to(room).emit('x')` is called by a socket in that room. Who receives `x`?
<details><summary>Answer</summary>Everyone in the room except the sender.</details>

**14.** Output — A handler throws a plain `Error` with no filter registered. What does the client see?
<details><summary>Answer</summary>Nothing useful — there is no status code channel, and without a filter the error is not delivered as a message.</details>

**15.** Output — Three replicas, no Redis adapter, 300 connected users evenly spread. A broadcast reaches how many?
<details><summary>Answer</summary>About 100 — only those on the emitting instance.</details>

**16.** Debug — A gateway accepts connections from any origin though the API's CORS is restricted. Cause?
<details><summary>Answer</summary>Gateway CORS is separate; `app.enableCors()` does not cover the WebSocket handshake. Configure `cors` in the gateway options.</details>

**17.** Debug — Users see their own messages twice. Which call is wrong?
<details><summary>Answer</summary>`this.server.to(room).emit(...)` when the sender also renders optimistically. Use `client.to(room)` to exclude the sender.</details>

**18.** Debug — Real-time updates work locally and reach only some users in production. Diagnosis?
<details><summary>Answer</summary>Multiple replicas without the Redis adapter, so broadcasts do not cross processes.</details>

**19.** Debug — Connections fail intermittently behind a load balancer with `websocket` and `polling` enabled. Cause?
<details><summary>Answer</summary>Polling handshake requests are landing on different instances. Enable sticky sessions or restrict to the websocket transport.</details>

**20.** Debug — After a crash and restart, many users appear permanently online. What was missing?
<details><summary>Answer</summary>A TTL on presence keys. `handleDisconnect` never ran for those sockets, so nothing removed them.</details>

**21.** Code — Authenticate a socket at connection and reject cleanly.
<details><summary>Answer</summary>

```ts
async handleConnection(client: Socket) {
  try {
    const token = client.handshake.auth?.token;
    const payload = await this.jwt.verifyAsync(token);
    const user = await this.users.findActiveById(payload.sub);
    if (!user) throw new Error();
    client.data.user = user;
    await client.join(`tenant:${user.tenantId}`);
  } catch {
    client.emit('exception', { message: 'Unauthorized' });
    client.disconnect(true);
  }
}
```
</details>

**22.** Code — Authorize a room subscription.
<details><summary>Answer</summary>

```ts
@SubscribeMessage('task:subscribe')
async subscribe(@MessageBody() dto: SubscribeDto, @ConnectedSocket() client: Socket) {
  const task = await this.tasks.findAccessible(dto.taskId, client.data.user);
  if (!task) throw new WsException('Task not found');
  await client.join(`task:${task.id}`);      // namespaced so ids cannot collide
  return { ok: true };
}
```
</details>

**23.** Code — Install the Redis adapter so broadcasts cross replicas.
<details><summary>Answer</summary>

```ts
export class RedisIoAdapter extends IoAdapter {
  constructor(app: INestApplication, private pub: RedisClientType, private sub: RedisClientType) {
    super(app);
  }
  createIOServer(port: number, options?: ServerOptions): Server {
    const server = super.createIOServer(port, options) as Server;
    server.adapter(createAdapter(this.pub, this.sub));
    return server;
  }
}
// main.ts
app.useWebSocketAdapter(new RedisIoAdapter(app, pubClient, subClient));
```
</details>

**24.** Scenario — A teammate proposes sending every database change to all connected clients and letting them filter. Evaluate it.
<details><summary>Answer</summary>It is a data leak, not just an inefficiency. Client-side filtering means the data has already crossed the wire, so anyone opening a developer console sees every other tenant's records regardless of what the UI renders. The bandwidth cost is real too — every client pays for every write in the system — but the security problem is disqualifying on its own. Broadcast to rooms scoped by tenant and resource, with membership authorized at join time, so the server decides who receives what. The general rule for real-time is the same as for REST: filtering is an authorization decision and must happen where the data lives, never where it is displayed.</details>

**25.** Scenario — Your gateway must support 50,000 concurrent connections across replicas, and users expect messages to survive a brief disconnect. How do you design it?
<details><summary>Answer</summary>Two separate problems. Fan-out across replicas is the Redis adapter's job, and at that connection count the adapter's own traffic matters — broadcast to narrow rooms rather than to a whole namespace, since every cross-instance broadcast costs a Redis round trip per instance. Delivery across a disconnect is not something a socket layer solves: an in-flight message to a dropped connection is lost, so durability needs a per-user inbox the client reconciles on reconnect, typically a stream or list with a cursor the client sends on connect to fetch what it missed. Treat the socket as a low-latency notification channel over a durable store rather than as the store itself. Also size the deployment by memory per connection and file descriptors, and force `transports: ['websocket']` so you are not paying for polling at that scale.</details>

---

## Chapter Summary

- A gateway maps messages to methods like a controller maps routes, but its CORS is configured separately — HTTP CORS does not cover the handshake.
- Authenticate once at connection and attach the principal to `client.data`; sending the token in a query string puts credentials into every proxy log.
- A socket authenticated once outlives token expiry, logout, and bans, so long-lived connections need re-validation or a revocation channel.
- Room names are an authorization boundary: letting a client choose one is a subscription to anything it can guess.
- Without the Redis adapter, broadcasts reach only one process — the single most common way real-time features work in development and fail in production.
- Connection-keyed state needs a reaping strategy, because `handleDisconnect` is exactly the path that does not run when a process dies.

---

## Practice Exercises

1. Configure restrictive HTTP CORS and connect a gateway from another origin to confirm they are independent.
2. Track presence as a boolean, open two tabs, close one, and observe the incorrect offline state.
3. Run two instances without the Redis adapter and measure how many clients receive a broadcast.
4. Let a client choose its own room name, then subscribe to another tenant's stream.
5. Kill a process without a clean shutdown and observe presence entries that never expire.

---

## Mini Project: A Collaborative Task Board

Build real-time collaboration that is correct across replicas.

**Requirements:**

- Handshake authentication with disconnect on failure, and expiry-driven disconnection of stale sockets.
- Rooms namespaced by tenant and task, with membership authorized on join.
- Redis adapter installed, proven by a test that broadcasts from one instance and receives on another.
- Presence as a Redis set with a TTL, correct for multiple tabs and after an unclean shutdown.
- A reconnect path where the client sends a cursor and receives missed events from a durable store.

---

**Next:** [Chapter 26 — Microservices and Transports →](26-microservices.md)
