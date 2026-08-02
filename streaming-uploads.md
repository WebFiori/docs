# Streaming Uploads

In this page:
* [Introduction](#introduction)
* [When to Use](#when-to-use)
* [Basic Usage](#basic-usage)
* [Filename Resolution](#filename-resolution)
* [Size Limits](#size-limits)
* [Stream Processors](#stream-processors)
* [Callbacks](#callbacks)
* [Frontend Example](#frontend-example)
* [Full API Example](#full-api-example)

## Introduction

[`StreamingUploader`](https://webfiori.com/docs/WebFiori/File/StreamingUploader) receives a single file from the raw HTTP body (`php://input`) in constant memory. Unlike `FileUploader` which works with `$_FILES` and multipart form data, this class reads binary data directly from the input stream — no temp file overhead, no memory spikes.

It extends [`AbstractUploader`](https://webfiori.com/docs/WebFiori/File/AbstractUploader), so all shared features (extension filtering, size limits, callbacks, stream processors) are available. See [Uploading Files](uploading-files.md#shared-features) for details on those.

## When to Use

- JavaScript `fetch()` or `XMLHttpRequest` sending a file as raw body
- Mobile apps uploading binary data directly
- Large single-file uploads where you want constant memory usage
- When you need to process the stream during upload (hashing, encryption)

If you need multipart form uploads, use [`FileUploader`](uploading-files.md). If you need chunked resume support, use [`ResumableUploader`](resumable-uploads.md).

## Basic Usage

``` php
use WebFiori\File\StreamingUploader;

$uploader = new StreamingUploader('/home/files/uploads', ['mp4', 'mov', 'avi']);
$file = $uploader->receive('user-video.mp4');

// $file is an UploadedFile instance
echo $file->getName();          // user-video.mp4
echo $file->isUploaded();       // true
echo $file->getAbsolutePath();  // /home/files/uploads/user-video.mp4
```

The `receive()` method:
1. Resolves the filename (from parameter, headers, or default)
2. Sanitizes the filename
3. Validates extension and size
4. Fires the before-upload callback
5. Reads from `php://input` and writes to disk
6. Fires the after-upload callback
7. Returns an `UploadedFile` instance

If any validation fails, a `FileException` is thrown.

## Filename Resolution

The filename is resolved in this order:

1. **Explicit parameter**: `$uploader->receive('my-file.pdf')`
2. **`X-Filename` header**: The client sends `X-Filename: my-file.pdf`
3. **`Content-Disposition` header**: `Content-Disposition: attachment; filename="my-file.pdf"`
4. **Default**: `upload.bin`

``` php
// Let the client specify the filename via header
$file = $uploader->receive(); // reads from X-Filename or Content-Disposition
```

## Size Limits

Size is checked in two stages:

1. **Before reading**: If a `Content-Length` header is present and exceeds the limit, the upload is rejected immediately.
2. **During reading**: Bytes are counted as they stream in. If the limit is exceeded mid-stream, the partial file is deleted and an exception is thrown.

``` php
$uploader->setMaxFileSize(100 * 1024 * 1024); // 100MB

try {
    $file = $uploader->receive('large-file.zip');
} catch (FileException $e) {
    // "File exceeds size limit."
}
```

## Stream Processors

Process data as it streams in — useful for hash verification, encryption, or transformations:

``` php
use Generator;
use WebFiori\File\StreamingUploader;

$uploader = new StreamingUploader('/home/files/uploads');
$checksum = null;

$uploader->setStreamProcessor(function (Generator $chunks, string $destPath) use (&$checksum) {
    $hash = hash_init('sha256');
    $dest = fopen($destPath, 'wb');

    foreach ($chunks as $chunk) {
        hash_update($hash, $chunk);
        fwrite($dest, $chunk);
    }

    fclose($dest);
    $checksum = hash_final($hash);
});

$file = $uploader->receive('document.pdf');
echo "SHA-256: $checksum";
```

When no stream processor is set, the uploader writes directly using `FileStream::writeFromStream()`.

## Callbacks

``` php
$uploader->setOnBeforeUpload(function (array $fileInfo): bool {
    // $fileInfo contains 'name' and 'upload-path'
    if (isBlacklisted($fileInfo['name'])) {
        return false; // reject
    }
    return true;
});

$uploader->setOnAfterUpload(function (UploadedFile $file): void {
    // Trigger async processing, notify user, etc.
    dispatch(new ProcessUploadedFile($file->getAbsolutePath()));
});
```

## Frontend Example

JavaScript client sending a file as raw binary:

``` javascript
const fileInput = document.querySelector('input[type="file"]');
const file = fileInput.files[0];

const response = await fetch('/api/upload', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/octet-stream',
        'X-Filename': file.name
    },
    body: file
});

const result = await response.json();
console.log(result);
```

## Full API Example

A complete web service that accepts streaming uploads using the `#[Consumes]` annotation:

``` php
use WebFiori\File\Exceptions\FileException;
use WebFiori\File\StreamingUploader;
use WebFiori\Http\Annotations\AllowAnonymous;
use WebFiori\Http\Annotations\Consumes;
use WebFiori\Http\Annotations\PostMapping;
use WebFiori\Http\Annotations\ResponseBody;
use WebFiori\Http\Annotations\RestController;
use WebFiori\Http\MediaType;
use WebFiori\Http\ResponseEntity;
use WebFiori\Http\WebService;

#[RestController('uploads', 'File upload service')]
class UploadService extends WebService {

    #[PostMapping]
    #[Consumes(MediaType::OCTET_STREAM)]
    #[ResponseBody]
    #[AllowAnonymous]
    public function upload(): ResponseEntity {
        $uploader = new StreamingUploader('/home/files/uploads', ['pdf', 'docx', 'xlsx']);
        $uploader->setMaxFileSize(50 * 1024 * 1024); // 50MB

        try {
            $file = $uploader->receive(); // filename from headers

            return ResponseEntity::created([
                'name' => $file->getName(),
                'size' => filesize($file->getAbsolutePath()),
                'mime' => $file->getMIME(),
            ]);
        } catch (FileException $e) {
            return ResponseEntity::unprocessableEntity(['error' => $e->getMessage()]);
        }
    }
}
```

The `#[Consumes(MediaType::OCTET_STREAM)]` annotation tells the framework to:
1. Accept `application/octet-stream` as a valid content type for this method
2. Skip parameter filtering (since binary data cannot be parsed as form fields)
3. Reject other content types (form-urlencoded, JSON, etc.) with 415

Without `#[Consumes]`, requests with `application/octet-stream` would be rejected with 415 Unsupported Media Type before reaching your code.

## Related Topics

* [Uploading Files](uploading-files.md) — Overview and `FileUploader` (multipart form uploads)
* [Resumable Uploads](resumable-uploads.md) — Chunked uploads with resume support
* [Web Services](web-services.md) — Create upload APIs
