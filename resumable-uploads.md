# Resumable Uploads

In this page:
* [Introduction](#introduction)
* [When to Use](#when-to-use)
* [How It Works](#how-it-works)
* [Basic Usage](#basic-usage)
* [Checking Offset (Resume)](#checking-offset-resume)
* [Canceling an Upload](#canceling-an-upload)
* [Cleaning Up Stale Partials](#cleaning-up-stale-partials)
* [Custom Partial Directory](#custom-partial-directory)
* [Callbacks and Stream Processors](#callbacks-and-stream-processors)
* [Frontend Example](#frontend-example)
* [Full API Example](#full-api-example)

## Introduction

[`ResumableUploader`](https://webfiori.com/docs/WebFiori/File/ResumableUploader) handles chunked file uploads with resume-on-failure support. Each chunk is a separate HTTP request. If the connection drops, the client can query the server for the current byte offset and resume from where it left off.

No database or session storage is needed — the partial file's size on disk serves as the authoritative byte offset.

It extends [`AbstractUploader`](https://webfiori.com/docs/WebFiori/File/AbstractUploader), so all shared features (extension filtering, size limits, callbacks, stream processors) are available. See [Uploading Files](uploading-files.md#shared-features) for details.

## When to Use

- Large file uploads over unreliable networks (mobile, satellite, poor WiFi)
- Files that are too large for a single HTTP request timeout
- When users need progress indicators and the ability to pause/resume
- When server-side memory constraints prevent loading entire files

If the network is reliable and files are small, [`StreamingUploader`](streaming-uploads.md) is simpler. For standard HTML forms, use [`FileUploader`](uploading-files.md).

## How It Works

1. Client generates a unique **upload ID** (e.g., UUID) for the session
2. Client splits the file into chunks and sends each as a separate request
3. Server appends each chunk to a partial file in a `.partial/` subdirectory
4. On failure, client asks the server for the current offset and resumes
5. On the final chunk, the server moves the partial file to the upload directory

```
Client                              Server
  |                                    |
  |--- Chunk 1 (bytes 0-8191) ------->|  append to .partial/id_file.dat
  |<-- { offset: 8192 } --------------|
  |                                    |
  |--- Chunk 2 (bytes 8192-16383) --->|  append
  |<-- { offset: 16384 } -------------|
  |                                    |
  |    *** connection drops ***        |
  |                                    |
  |--- GET offset? ------------------->|  check filesize
  |<-- { offset: 16384 } -------------|
  |                                    |
  |--- Chunk 3 (final) -------------->|  append + move to uploads/
  |<-- { complete: true, file } ------|
```

## Basic Usage

``` php
use WebFiori\File\ResumableUploader;

$uploader = new ResumableUploader('/home/files/uploads', ['mp4', 'zip']);

// Each request provides the upload ID, filename, and whether it's the last chunk
$result = $uploader->receiveChunk(
    uploadId: 'abc-123-def',
    filename: 'large-video.mp4',
    isLast: false
);

// $result structure:
// [
//     'offset'   => 8192,          // bytes received so far
//     'complete' => false,         // not done yet
//     'file'     => null           // only set when complete
// ]
```

On the final chunk:

``` php
$result = $uploader->receiveChunk('abc-123-def', 'large-video.mp4', true);

// [
//     'offset'   => 524288,
//     'complete' => true,
//     'file'     => UploadedFile instance
// ]
```

## Checking Offset (Resume)

When a client reconnects after a failure, it queries the current offset:

``` php
$uploader = new ResumableUploader('/home/files/uploads');
$offset = $uploader->getOffset('abc-123-def', 'large-video.mp4');

// Returns 0 if no partial file exists, otherwise the byte count
```

The client then skips to that offset and resumes sending.

## Canceling an Upload

Remove the partial file for a given session:

``` php
$uploader->cancel('abc-123-def', 'large-video.mp4');
```

## Cleaning Up Stale Partials

Remove partial files older than a given age. Useful as a scheduled task:

``` php
$uploader = new ResumableUploader('/home/files/uploads');
$removed = $uploader->cleanStale(3600); // remove partials older than 1 hour
echo "$removed stale files cleaned up";
```

## Custom Partial Directory

By default, partial files are stored in `.partial/` inside the upload directory. You can change this:

``` php
$uploader->setPartialDir('/tmp/upload-partials');
```

## Callbacks and Stream Processors

The before-upload callback fires only on the **first chunk** of a session. The after-upload callback fires when the final chunk completes.

``` php
$uploader->setOnBeforeUpload(function (array $fileInfo): bool {
    // $fileInfo includes 'name', 'upload-path', and 'upload-id'
    return isAllowedUser($fileInfo['upload-id']);
});

$uploader->setOnAfterUpload(function (UploadedFile $file): void {
    notifyUser('Upload complete: ' . $file->getName());
});
```

Stream processors run during finalization — the partial file is read through the processor and written to the final destination:

``` php
$uploader->setStreamProcessor(function (Generator $chunks, string $destPath): void {
    $dest = fopen($destPath, 'wb');
    foreach ($chunks as $chunk) {
        fwrite($dest, $chunk);
    }
    fclose($dest);
});
```

## Frontend Example

JavaScript client with chunked upload and resume:

``` javascript
const CHUNK_SIZE = 64 * 1024; // 64KB chunks
const uploadId = crypto.randomUUID();

async function uploadFile(file) {
    let offset = await getOffset(uploadId, file.name);

    while (offset < file.size) {
        const isLast = (offset + CHUNK_SIZE) >= file.size;
        const chunk = file.slice(offset, offset + CHUNK_SIZE);

        const response = await fetch('/api/upload/chunk', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/octet-stream',
                'X-Upload-Id': uploadId,
                'X-Filename': file.name,
                'X-Is-Last': isLast ? '1' : '0'
            },
            body: chunk
        });

        const result = await response.json();
        offset = result.offset;

        if (result.complete) {
            console.log('Upload complete:', result.file);
            return;
        }
    }
}

async function getOffset(uploadId, filename) {
    const response = await fetch(`/api/upload/offset?id=${uploadId}&name=${filename}`);
    const data = await response.json();
    return data.offset;
}
```

## Full API Example

A backend API handling chunk uploads, offset queries, and cancellation:

``` php
use WebFiori\File\Exceptions\FileException;
use WebFiori\File\ResumableUploader;

$uploader = new ResumableUploader('/home/files/uploads', ['mp4', 'zip', 'pdf']);
$uploader->setMaxFileSize(500 * 1024 * 1024); // 500MB

$method = $_SERVER['REQUEST_METHOD'];
$path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);

if ($method === 'GET' && $path === '/api/upload/offset') {
    // Resume check
    $uploadId = $_GET['id'] ?? '';
    $filename = $_GET['name'] ?? '';
    $offset = $uploader->getOffset($uploadId, $filename);

    header('Content-Type: application/json');
    echo json_encode(['offset' => $offset]);

} elseif ($method === 'POST' && $path === '/api/upload/chunk') {
    // Receive chunk
    $uploadId = $_SERVER['HTTP_X_UPLOAD_ID'] ?? '';
    $filename = $_SERVER['HTTP_X_FILENAME'] ?? null;
    $isLast = ($_SERVER['HTTP_X_IS_LAST'] ?? '0') === '1';

    try {
        $result = $uploader->receiveChunk($uploadId, $filename, $isLast);

        header('Content-Type: application/json');
        http_response_code($result['complete'] ? 201 : 200);
        echo json_encode([
            'offset' => $result['offset'],
            'complete' => $result['complete'],
            'file' => $result['file'] ? $result['file']->getName() : null,
        ]);
    } catch (FileException $e) {
        http_response_code(422);
        header('Content-Type: application/json');
        echo json_encode(['error' => $e->getMessage()]);
    }

} elseif ($method === 'DELETE' && $path === '/api/upload/cancel') {
    // Cancel upload
    $uploadId = $_GET['id'] ?? '';
    $filename = $_GET['name'] ?? '';
    $uploader->cancel($uploadId, $filename);

    http_response_code(204);
}
```

## Related Topics

* [Uploading Files](uploading-files.md) — Overview and `FileUploader` (multipart form uploads)
* [Streaming Uploads](streaming-uploads.md) — Single-shot raw body uploads
* [Background Tasks](background-tasks.md) — Schedule stale partial cleanup
* [Web Services](web-services.md) — Create upload APIs
