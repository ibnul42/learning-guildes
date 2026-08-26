# Chapter 31 — File Uploads and Streaming

## Learning Objectives

- Accept uploads with Multer interceptors and validate them with `ParseFilePipe`
- Explain why the reported MIME type is not validation, and check magic bytes instead
- Stream large files out with `StreamableFile` without buffering them in memory
- Stream uploads to object storage rather than through your process
- Serve user-supplied files without creating stored XSS or path traversal

---

## 31.1 Accepting an Upload

On Express, uploads go through Multer:

```ts
@Post('avatar')
@UseInterceptors(FileInterceptor('file'))
uploadAvatar(@UploadedFile() file: Express.Multer.File) {
  return this.storage.save(file);
}
```

```ts
@UseInterceptors(FilesInterceptor('files', 10))          // several under one field
@UseInterceptors(FileFieldsInterceptor([                 // several named fields
  { name: 'avatar', maxCount: 1 },
  { name: 'documents', maxCount: 5 },
]))
```

**The default stores the whole file in memory.** A 500 MB upload becomes a 500 MB buffer, and ten concurrent uploads exhaust the heap. Configure a destination or a size limit explicitly:

```ts
@UseInterceptors(
  FileInterceptor('file', {
    storage: diskStorage({
      destination: '/tmp/uploads',
      filename: (_req, _file, cb) => cb(null, `${randomUUID()}`),   // never trust the client's name
    }),
    limits: { fileSize: 5 * 1024 * 1024, files: 1 },
  }),
)
```

> ⚠️ **Gotcha** — `limits.fileSize` is enforced by Multer *while streaming*, so it stops the transfer rather than accepting it and rejecting afterwards. A `MaxFileSizeValidator` in a pipe runs **after** the file has been fully received, so it protects your storage but not your bandwidth or memory. Use both: the limit for defence, the validator for the error message.

Fastify does not use Multer at all — it needs `@fastify/multipart`, and the handler code differs. This is the platform abstraction leaking again (→ [Ch 1](01-what-nestjs-is.md)).

---

## 31.2 Validation Is Not What It Looks Like

`ParseFilePipe` composes validators:

```ts
@Post('avatar')
@UseInterceptors(FileInterceptor('file'))
upload(
  @UploadedFile(
    new ParseFilePipe({
      validators: [
        new MaxFileSizeValidator({ maxSize: 5 * 1024 * 1024 }),
        new FileTypeValidator({ fileType: /^image\/(jpeg|png|webp)$/ }),
      ],
      fileIsRequired: true,
    }),
  )
  file: Express.Multer.File,
) {}
```

**`FileTypeValidator` checks the `Content-Type` the client sent.** That header is chosen by the client and can say anything. A PHP script, an HTML page, or an executable uploads happily as `image/png`:

```bash
curl -F 'file=@shell.html;type=image/png' https://api.example.com/avatar   # passes
```

Real validation reads the file's own leading bytes:

```ts
import { fileTypeFromBuffer } from 'file-type';

const detected = await fileTypeFromBuffer(file.buffer);
if (!detected || !['image/jpeg', 'image/png', 'image/webp'].includes(detected.mime)) {
  throw new UnsupportedMediaTypeException('File content is not a supported image');
}
```

Two more rules that matter as much:

**Never use the client's filename.** `../../etc/passwd` and `..\\..\\config` are path traversal, and a filename ending `.html` served from your origin is stored XSS. Generate your own name and store the original only as a display label.

**Serve user content from a separate origin.** Even with correct headers, a file served from your API's origin runs in that origin's context if a browser can be persuaded to render it. A distinct domain or bucket makes an injected script harmless to your session cookies (→ [Ch 23](23-security.md)).

---

## 31.3 Streaming Downloads

Reading a file into memory to return it does not scale:

```ts
// ❌ A 2 GB file is a 2 GB buffer, and ten concurrent downloads is 20 GB
@Get(':id')
async download(@Param('id') id: string) {
  return await readFile(path);
}

// ✅ Constant memory regardless of size
@Get(':id')
download(@Param('id') id: string, @Res({ passthrough: true }) res: Response): StreamableFile {
  const stream = createReadStream(path);
  res.set({
    'Content-Type': 'application/pdf',
    'Content-Disposition': `attachment; filename="report.pdf"`,
    // Prevents a browser from re-interpreting the type regardless of extension
    'X-Content-Type-Options': 'nosniff',
  });
  return new StreamableFile(stream);
}
```

`StreamableFile` pipes the stream to the response, so memory stays flat.

`Content-Disposition` deserves care. `attachment` forces a download; `inline` renders in the browser — and `inline` on user-uploaded content is how stored XSS becomes exploitable. The filename must also be escaped, since a quote in it can inject additional header directives.

> 💼 **Interview** — "How do you serve a large file from a Node API?" The expected answer is streaming rather than buffering. The follow-up worth pre-empting: for genuinely large files the best answer is not to serve it at all — issue a pre-signed URL and let object storage do it, so your process never touches the bytes.

---

## 31.4 Not Touching the Bytes

For anything of size, the strongest design keeps files out of your application entirely.

**Pre-signed upload URLs.** The client requests permission, uploads directly to S3, and tells you when it is done:

```ts
@Post('uploads/presign')
async presign(@Body() dto: PresignDto, @CurrentUser('id') userId: string) {
  const key = `uploads/${userId}/${randomUUID()}`;

  const url = await getSignedUrl(
    this.s3,
    new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      ContentType: dto.contentType,
      // Enforced by S3 at upload time — the client cannot exceed it
      ContentLength: dto.size,
    }),
    { expiresIn: 300 },
  );

  return { url, key };
}
```

Your process handles two small JSON requests instead of the file. Bandwidth, memory, and request duration all stop being your problem.

The trade-off is that **you never see the content**, so validation must happen afterwards — a storage event triggering a job that inspects the object and quarantines it if it is not what was declared (→ [Ch 27](27-queues-jobs.md)). Until that job runs, treat the object as untrusted.

**Pre-signed download URLs** are the same idea in reverse, and they also solve authorization for large files: the URL is short-lived and scoped to one object, so you authorize once and let storage serve.

---

## 31.5 Backpressure

Streaming only helps if backpressure is respected. `pipe` and `pipeline` handle it; manual loops usually do not:

```ts
// ❌ Ignores whether the destination is keeping up — memory grows unbounded
source.on('data', (chunk) => destination.write(chunk));

// ✅ pipeline propagates backpressure AND cleans up on error
import { pipeline } from 'node:stream/promises';
await pipeline(source, transform, destination);
```

`pipeline` also destroys every stream in the chain if one fails, which plain `.pipe()` does not — an error mid-transfer with `.pipe()` leaves file handles and sockets open.

**Streaming does not make CPU work free.** Image resizing or PDF generation on a streamed upload still blocks the event loop, so that work belongs in a queue with a worker process (→ [Ch 27](27-queues-jobs.md)).

---

## 31.6 Common Mistakes and Best Practices

**Mistake: memory storage with no size limit.** Concurrent uploads exhaust the heap.

**Mistake: trusting `FileTypeValidator`.** It reads a client-supplied header, not the file.

**Mistake: using the client's filename.** Path traversal, and `.html` on your origin is stored XSS.

**Mistake: serving user uploads from the API origin.** A rendered file executes in your origin's context.

**Mistake: `Content-Disposition: inline` for user content.** It is what makes stored XSS exploitable.

**Mistake: reading a file into memory to return it.** Memory scales with file size times concurrency.

**Mistake: manual `data`/`write` loops.** Backpressure is ignored and errors leak handles.

**Best practice: enforce size at the transport with `limits`, and validate content with magic bytes.** One protects bandwidth, the other protects correctness.

**Best practice: pre-sign for anything large.** Your process should not be a file proxy.

**Best practice: use `pipeline` rather than `pipe`.** It propagates backpressure and destroys the chain on error.

---

## 31.7 Real-World Example: Pre-Signed Upload With Post-Upload Verification

The production shape: the client uploads directly to storage, your API only issues and records permission, and a background job verifies the object before it is usable.

```ts
// src/uploads/uploads.service.ts
@Injectable()
export class UploadsService {
  private static readonly ALLOWED = new Map<string, string[]>([
    ['image/jpeg', ['jpg', 'jpeg']],
    ['image/png', ['png']],
    ['image/webp', ['webp']],
    ['application/pdf', ['pdf']],
  ]);
  private static readonly MAX_BYTES = 25 * 1024 * 1024;

  constructor(
    private readonly s3: S3Client,
    private readonly repo: UploadRepository,
    @InjectQueue('uploads') private readonly queue: Queue,
    @Inject(storageConfig.KEY) private readonly config: ConfigType<typeof storageConfig>,
  ) {}

  async presign(dto: PresignDto, userId: string): Promise<PresignResult> {
    if (!UploadsService.ALLOWED.has(dto.contentType)) {
      throw new UnsupportedMediaTypeException(`${dto.contentType} is not accepted`);
    }
    if (dto.size > UploadsService.MAX_BYTES) {
      throw new PayloadTooLargeException(`Maximum size is ${UploadsService.MAX_BYTES} bytes`);
    }

    // The key is generated, never derived from a client filename. The user id
    // is a path segment so an object cannot be written outside their prefix.
    const key = `uploads/${userId}/${randomUUID()}`;

    // Recorded as `pending` BEFORE the URL is issued: an upload we have no
    // record of is an orphaned object nothing will ever clean up.
    const upload = await this.repo.create({
      key,
      ownerId: userId,
      declaredType: dto.contentType,
      declaredSize: dto.size,
      originalName: dto.filename.slice(0, 255),     // stored as a label only
      status: 'pending',
    });

    const url = await getSignedUrl(
      this.s3,
      new PutObjectCommand({
        Bucket: this.config.bucket,
        Key: key,
        ContentType: dto.contentType,
        ContentLength: dto.size,      // S3 rejects a mismatch — a real limit
      }),
      { expiresIn: 300 },
    );

    return { uploadId: upload.id, url, key };
  }

  /** Called by the client after the PUT succeeds, or by a bucket event. */
  async confirm(uploadId: string, userId: string): Promise<Upload> {
    const upload = await this.repo.findOwned(uploadId, userId);
    if (!upload) throw new NotFoundException();

    await this.repo.update(upload.id, { status: 'verifying' });

    await this.queue.add(
      'verify',
      { uploadId: upload.id },
      { jobId: `verify:${upload.id}`, attempts: 3 },   // idempotent enqueue (→ Ch 27)
    );
    return upload;
  }
}
```

```ts
// src/uploads/verify.processor.ts
@Processor('uploads', { concurrency: 4 })
export class VerifyUploadProcessor extends WorkerHost {
  async process(job: Job<{ uploadId: string }>): Promise<void> {
    const upload = await this.repo.findById(job.data.uploadId);
    if (!upload) throw new UnrecoverableError('Upload record no longer exists');

    const head = await this.s3.send(
      new HeadObjectCommand({ Bucket: this.config.bucket, Key: upload.key }),
    );

    // The declared size was enforced by S3, but re-check: it is one call and
    // it catches a bucket policy that was loosened later.
    if ((head.ContentLength ?? 0) > UploadsService.MAX_BYTES) {
      await this.quarantine(upload, 'exceeds maximum size');
      return;
    }

    // Read only the first bytes. Magic-byte detection needs a few hundred,
    // and downloading a 25 MB file to read its header would be absurd.
    const range = await this.s3.send(
      new GetObjectCommand({
        Bucket: this.config.bucket,
        Key: upload.key,
        Range: 'bytes=0-4095',
      }),
    );
    const header = Buffer.from(await range.Body!.transformToByteArray());

    const detected = await fileTypeFromBuffer(header);

    // The authoritative check: what the bytes ARE, not what the client said.
    if (!detected || detected.mime !== upload.declaredType) {
      await this.quarantine(
        upload,
        `content is ${detected?.mime ?? 'unrecognised'}, declared ${upload.declaredType}`,
      );
      return;
    }

    await this.repo.update(upload.id, { status: 'ready', verifiedType: detected.mime });
  }

  private async quarantine(upload: Upload, reason: string): Promise<void> {
    // Move rather than delete: a false positive should be recoverable, and
    // the object is evidence if the upload was hostile.
    await this.s3.send(
      new CopyObjectCommand({
        Bucket: this.config.quarantineBucket,
        CopySource: `${this.config.bucket}/${upload.key}`,
        Key: upload.key,
      }),
    );
    await this.s3.send(
      new DeleteObjectCommand({ Bucket: this.config.bucket, Key: upload.key }),
    );
    await this.repo.update(upload.id, { status: 'rejected', rejectionReason: reason });
  }
}
```

```ts
// src/uploads/uploads.controller.ts
@Controller('uploads')
export class UploadsController {
  @Get(':id/url')
  async downloadUrl(@Param('id') id: string, @CurrentUser('id') userId: string) {
    const upload = await this.uploads.findOwned(id, userId);

    // Only verified objects are downloadable. A pending or rejected upload
    // must not be retrievable, or verification is decorative.
    if (!upload || upload.status !== 'ready') throw new NotFoundException();

    const url = await getSignedUrl(
      this.s3,
      new GetObjectCommand({
        Bucket: this.config.bucket,
        Key: upload.key,
        // Forces download and pins the type, so a file that slipped through
        // still cannot be rendered as HTML in the browser.
        ResponseContentDisposition: `attachment; filename="${sanitise(upload.originalName)}"`,
        ResponseContentType: upload.verifiedType,
      }),
      { expiresIn: 60 },
    );
    return { url };
  }
}
```

The decision worth defending is recording the upload as `pending` *before* issuing the pre-signed URL. Creating the record on confirmation is the simpler flow and it produces objects nobody knows about: the client uploads successfully and then crashes, loses connectivity, or simply never calls confirm, and the bucket accumulates files with no database row, no owner, and no lifecycle. Nothing will ever find them, because the only way to enumerate them is to list the bucket and diff it against the database — a reconciliation job that has to exist precisely because the record was created too late. Writing the row first means every object has a row from the moment it can exist, so a sweeper can find `pending` rows older than the URL's expiry and clean up deterministically. The general principle is that when you hand out permission to create something you will not observe, record the grant, not the outcome.

---

## 31.8 Questions

**1.** MCQ — Multer's default storage keeps the file: a) on disk b) in memory c) streamed d) in S3
<details><summary>Answer</summary>**b)** Which is why an explicit size limit or disk destination matters.</details>

**2.** MCQ — `FileTypeValidator` inspects: a) magic bytes b) the client-supplied MIME type c) the extension d) file size
<details><summary>Answer</summary>**b)** The header the client chose, so it is trivially spoofed.</details>

**3.** MCQ — `limits.fileSize` versus `MaxFileSizeValidator`: the limit is enforced: a) after receipt b) during streaming c) never d) by the browser
<details><summary>Answer</summary>**b)** So it protects bandwidth and memory; the validator only produces a nicer error afterwards.</details>

**4.** MCQ — `StreamableFile` gives you: a) faster reads b) constant memory c) compression d) caching
<details><summary>Answer</summary>**b)** Memory does not scale with file size.</details>

**5.** MCQ — Which propagates backpressure and cleans up on error? a) `.pipe()` b) a `data` handler c) `pipeline` d) `for await`
<details><summary>Answer</summary>**c)** `pipe` propagates backpressure but does not destroy the chain on error.</details>

**6.** MCQ — A pre-signed upload URL means your process handles: a) the whole file b) two small JSON requests c) half the file d) nothing at all
<details><summary>Answer</summary>**b)** The bytes go straight to storage.</details>

**7.** Short — Why is the reported MIME type not validation?
<details><summary>Answer</summary>It is a header the client sets, so any file can claim any type. Only the leading bytes identify the actual format.</details>

**8.** Short — Why must you never use the client's filename?
<details><summary>Answer</summary>It enables path traversal, and an `.html` name served from your origin turns an upload into stored XSS.</details>

**9.** Short — Why serve user uploads from a separate origin?
<details><summary>Answer</summary>A file rendered from your API's origin executes in that origin's context, so it can reach cookies and same-origin endpoints.</details>

**10.** Short — What do you give up with pre-signed uploads?
<details><summary>Answer</summary>Visibility of the content at upload time, so validation must happen afterwards and the object must be treated as untrusted until then.</details>

**11.** Short — Why record the upload before issuing the URL?
<details><summary>Answer</summary>Otherwise a client that uploads and never confirms leaves an object with no database row, which nothing can find without reconciling the whole bucket.</details>

**12.** Output — A 500 MB file is uploaded with default Multer settings. What happens to memory?
<details><summary>Answer</summary>A 500 MB buffer is allocated; a few concurrent uploads exhaust the heap.</details>

**13.** Output — `curl -F 'file=@shell.html;type=image/png'` against a route using `FileTypeValidator`. Accepted?
<details><summary>Answer</summary>Yes. The validator reads the declared type, which says `image/png`.</details>

**14.** Output — A 2 GB file returned with `await readFile(path)`. What is the memory cost for three concurrent requests?
<details><summary>Answer</summary>About 6 GB. Use `StreamableFile`.</details>

**15.** Output — `source.on('data', c => dest.write(c))` where the destination is slower than the source. What grows?
<details><summary>Answer</summary>Memory — writes are queued because backpressure is ignored.</details>

**16.** Debug — An uploaded HTML file executed script in a user's session. Which two decisions caused it?
<details><summary>Answer</summary>Trusting the declared MIME type, and serving the file from the API's own origin with a disposition that let it render.</details>

**17.** Debug — Files appear in the bucket with no corresponding database row. Cause?
<details><summary>Answer</summary>The record is created on confirmation rather than when the URL is issued, so any upload the client never confirms is orphaned.</details>

**18.** Debug — Uploads work on Express and every handler breaks after switching to Fastify. Why?
<details><summary>Answer</summary>Multer is Express-only; Fastify needs `@fastify/multipart` with a different handler API.</details>

**19.** Debug — A failed download leaves file handles open. Which stream call is responsible?
<details><summary>Answer</summary>`.pipe()`, which does not destroy the chain on error. Use `pipeline`.</details>

**20.** Debug — Image resizing on upload made every request slow, though the upload is streamed. Why?
<details><summary>Answer</summary>Resizing is CPU-bound and blocks the event loop regardless of how the bytes arrived. Move it to a queue with a worker process.</details>

**21.** Code — Validate an upload by content rather than by header.
<details><summary>Answer</summary>

```ts
const detected = await fileTypeFromBuffer(file.buffer.subarray(0, 4096));
const allowed = ['image/jpeg', 'image/png', 'image/webp'];
if (!detected || !allowed.includes(detected.mime)) {
  throw new UnsupportedMediaTypeException('File content is not a supported image');
}
```
</details>

**22.** Code — Stream a download with headers that cannot be re-interpreted.
<details><summary>Answer</summary>

```ts
@Get(':id')
download(@Param('id') id: string, @Res({ passthrough: true }) res: Response): StreamableFile {
  res.set({
    'Content-Type': 'application/pdf',
    'Content-Disposition': `attachment; filename="${sanitise(name)}"`,
    'X-Content-Type-Options': 'nosniff',
  });
  return new StreamableFile(createReadStream(path));
}
```
</details>

**23.** Code — Issue a pre-signed upload URL with an enforced size.
<details><summary>Answer</summary>

```ts
const url = await getSignedUrl(
  this.s3,
  new PutObjectCommand({
    Bucket: bucket,
    Key: `uploads/${userId}/${randomUUID()}`,
    ContentType: dto.contentType,
    ContentLength: dto.size,        // S3 rejects a mismatched upload
  }),
  { expiresIn: 300 },
);
```
</details>

**24.** Scenario — A teammate proposes accepting uploads through the API so files can be validated before they reach storage. Evaluate it.
<details><summary>Answer</summary>The instinct is sound and the cost is high. Proxying files means every upload occupies a request slot for its whole duration, memory scales with size times concurrency, and your autoscaling now tracks upload volume rather than API load — one user uploading large files can degrade the API for everyone. The validation benefit is also smaller than it appears: you can check magic bytes at the boundary, but anything deeper (a malformed image, embedded content) needs real processing that belongs in a worker anyway. Pre-sign, then verify asynchronously and keep the object unusable until verification passes. If there is a hard requirement that nothing unverified ever exists in the main bucket, upload to a quarantine bucket and promote on success — same architecture, stricter placement.</details>

**25.** Scenario — Users must upload multi-gigabyte video files over unreliable connections. What do you build?
<details><summary>Answer</summary>Multipart upload directly to object storage, not anything that passes through your API. The client requests an upload id, uploads parts independently with per-part pre-signed URLs, retries only failed parts, and calls a completion endpoint that assembles them — which makes a dropped connection cost one part rather than the whole file, and makes resumption possible. Your API issues small grants and records state; it never sees a byte. Set a lifecycle rule to abort incomplete multipart uploads after a few days, because unassembled parts are billed and invisible in normal bucket listings — this is the detail people miss and discover on an invoice. Verification and transcoding then run as queued jobs against the assembled object, with the file unusable until they pass.</details>

---

## Chapter Summary

- Multer buffers to memory by default, so an explicit `limits.fileSize` is the difference between a rejected upload and an exhausted heap.
- `FileTypeValidator` reads a client-supplied header; only magic-byte detection tells you what a file actually is.
- Never reuse a client filename and never serve user content from your API's origin — those two together are how an upload becomes stored XSS.
- `StreamableFile` keeps memory flat for downloads, and `pipeline` is what propagates backpressure and destroys the chain when something fails.
- For anything large, pre-signed URLs keep the bytes out of your process entirely, at the cost of validating after the fact.
- Record the upload when you issue the grant rather than when the client confirms, or unconfirmed objects become unfindable.

---

## Practice Exercises

1. Upload a 200 MB file with default Multer settings and watch the process memory.
2. Rename an HTML file to `.png`, declare it as `image/png`, and confirm `FileTypeValidator` accepts it.
3. Return a 1 GB file with `readFile` and then with `StreamableFile`, comparing memory.
4. Replace `.pipe()` with `pipeline`, kill the destination mid-transfer, and compare handle cleanup.
5. Implement pre-signed upload, abandon the confirm step, and write the sweeper that finds the orphan.

---

## Mini Project: A Document Service

Build uploads you would expose to untrusted users.

**Requirements:**

- Pre-signed uploads with server-enforced type and size, and a `pending` record written before the URL is issued.
- Asynchronous verification by magic bytes, quarantining anything whose content contradicts its declaration.
- Downloads via short-lived pre-signed URLs with forced `attachment` disposition and a pinned content type.
- A sweeper that reconciles `pending` rows older than the URL expiry and aborts incomplete multipart uploads.
- Tests covering: a spoofed MIME type, a traversal filename, an abandoned upload, and an oversized file.

---

**Next:** [Chapter 32 — Task Scheduling →](32-task-scheduling.md)
