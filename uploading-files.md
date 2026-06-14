# Uploading Files

In this page:
* [Introduction](#introduction)
* [Shared Features](#shared-features)
  * [Extension Filtering](#extension-filtering)
  * [Size Limits](#size-limits)
  * [Callbacks](#callbacks)
  * [Stream Processors](#stream-processors)
  * [Filename Sanitization](#filename-sanitization)
* [The Class `FileUploader`](#the-class-fileuploader)
  * [Uploading One File](#uploading-one-file)
  * [Uploading Multiple Files](#uploading-multiple-files)
  * [Getting Upload Results](#getting-upload-results)
* [Choosing the Right Uploader](#choosing-the-right-uploader)
* [Serving Uploaded Files](#serving-uploaded-files)

## Introduction

The framework provides three uploader classes for different use cases, all in the namespace `WebFiori\File`:

| Class | Use Case |
|-------|----------|
| [`FileUploader`](https://webfiori.com/docs/WebFiori/File/FileUploader) | Standard HTML form uploads (multipart/form-data) |
| [`StreamingUploader`](https://webfiori.com/docs/WebFiori/File/StreamingUploader) | Raw binary body uploads (`php://input`) in constant memory |
| [`ResumableUploader`](https://webfiori.com/docs/WebFiori/File/ResumableUploader) | Chunked uploads with pause/resume support |

All three extend [`AbstractUploader`](https://webfiori.com/docs/WebFiori/File/AbstractUploader) which provides shared functionality: extension filtering, size limits, callbacks, stream processors, and filename sanitization.

This page covers `FileUploader`. For the other two, see:
- [Streaming Uploads](streaming-uploads.md)
- [Resumable Uploads](resumable-uploads.md)

## Shared Features

These features are available on all three uploaders.

### Extension Filtering

Restrict which file types can be uploaded:

``` php
$uploader->addExts(['jpg', 'png', 'pdf']);
$uploader->addExt('docx');
$uploader->removeExt('png');

// Check current allowed types
$allowed = $uploader->getExts(); // ['jpg', 'pdf', 'docx']
```

If no extensions are added, all types are accepted.

### Size Limits

Set a custom maximum file size (in bytes) beyond PHP's `upload_max_filesize`:

``` php
$uploader->setMaxFileSize(10 * 1024 * 1024); // 10MB

// Check the limit
$limit = $uploader->getMaxFileSizeLimit(); // int or null
```

For `FileUploader`, you can also check PHP's configured limit:

``` php
$phpMax = FileUploader::getMaxFileSize(); // in KB
```

### Callbacks

Hook into the upload process for validation, logging, or post-processing:

``` php
// Before upload — return false to reject
$uploader->setOnBeforeUpload(function (array $fileInfo): bool {
    if (str_contains($fileInfo['name'], 'blocked')) {
        return false; // reject this file
    }
    return true;
});

// After upload — runs for each successfully uploaded file
$uploader->setOnAfterUpload(function (UploadedFile $file): void {
    log_info('Uploaded: ' . $file->getName() . ' to ' . $file->getDir());
});
```

### Stream Processors

Process file data during upload (e.g., hash verification, encryption, virus scanning):

``` php
$uploader->setStreamProcessor(function (Generator $chunks, string $destPath): void {
    $hash = hash_init('sha256');
    $dest = fopen($destPath, 'wb');

    foreach ($chunks as $chunk) {
        hash_update($hash, $chunk);
        fwrite($dest, $chunk);
    }

    fclose($dest);
    $checksum = hash_final($hash);
    // Store or verify $checksum as needed
});
```

When a stream processor is set, the uploader pipes raw file data through it instead of using the default file move.

### Filename Sanitization

All uploaders automatically sanitize filenames using `AbstractUploader::sanitizeFilename()`. This removes path traversal attempts, null bytes, and special characters — replacing them with underscores.

## The Class `FileUploader`

[`FileUploader`](https://webfiori.com/docs/WebFiori/File/FileUploader) handles standard HTML form uploads using `$_FILES`. It supports single and multiple file uploads.

### Uploading One File

Backend:

``` php
use WebFiori\File\FileUploader;

// Constructor accepts upload path and allowed types directly
$uploader = new FileUploader('/home/files/uploads', ['doc', 'docx', 'pdf']);

// Set the name of the file input element
$uploader->setAssociatedFileName('user-file');

// Optional: set size limit and callbacks
$uploader->setMaxFileSize(5 * 1024 * 1024); // 5MB

// Upload. Pass true to replace existing files.
$uploader->upload(true);

// Return JSON status
echo $uploader;
```

Frontend (HTML form):

``` html
<form method="post" action="apis/upload-files" enctype="multipart/form-data">
    <input type="file" name="user-file">
    <button type="submit">Upload</button>
</form>
```

> **Note:** You can skip setting the associated file name in the backend by adding a hidden input: `<input type="hidden" name="file" value="user-file">`. The uploader reads this automatically.

### Uploading Multiple Files

The backend code is identical. The only difference is in the HTML — add `multiple` and use array syntax for the name:

``` html
<form method="post" action="apis/upload-files" enctype="multipart/form-data">
    <input type="file" name="user-file[]" multiple>
    <button type="submit">Upload</button>
</form>
```

### Getting Upload Results

The `upload()` method returns an array of associative arrays with status info for each file:

``` php
$results = $uploader->upload();

foreach ($results as $fileInfo) {
    echo $fileInfo['name'];       // filename
    echo $fileInfo['size'];       // size in bytes
    echo $fileInfo['upload-path'];// server path
    echo $fileInfo['mime'];       // MIME type
    echo $fileInfo['uploaded'];   // true/false
    echo $fileInfo['upload-error']; // error message or empty
    echo $fileInfo['is-exist'];   // file already existed?
    echo $fileInfo['is-replace']; // was it replaced?
}
```

To get results as objects instead:

``` php
$files = $uploader->uploadAsFileObj();

foreach ($files as $file) {
    // $file is an instance of UploadedFile (extends File)
    echo $file->getName();
    echo $file->isUploaded() ? 'success' : $file->getUploadError();
}
```

You can also retrieve file info after uploading:

``` php
$uploader->upload();
$files = $uploader->getFiles();       // array of UploadedFile objects
$files = $uploader->getFiles(false);  // array of associative arrays
```

## Choosing the Right Uploader

| Scenario | Uploader |
|----------|----------|
| HTML form with `<input type="file">` | `FileUploader` |
| JavaScript `fetch()` sending raw binary body | [`StreamingUploader`](streaming-uploads.md) |
| Large files over unreliable networks, need pause/resume | [`ResumableUploader`](resumable-uploads.md) |
| Files that don't fit in PHP memory | `StreamingUploader` or `ResumableUploader` |

## Serving Uploaded Files

After uploading, you can serve files back to clients using the `ResponseEmitter` interface:

``` php
use WebFiori\File\FileStream;

// Serve a large file with constant memory usage
$stream = new FileStream('/home/files/uploads/report.pdf');
$stream->serve(true); // true = download dialog (attachment)
```

For files already loaded in memory:

``` php
use WebFiori\File\File;

$file = new File('report.pdf', '/home/files/uploads');
$file->read();
$file->view(true); // true = as attachment
```

The `ResponseEmitter` interface (`setHeader()`, `setStatusCode()`, `sendBody()`) decouples file serving from any specific HTTP framework. The library ships with `DefaultEmitter` (raw PHP) and `WebFioriEmitter` (integrates with `WebFiori\Http\Response`).

To use a custom emitter:

``` php
$stream = new FileStream('/path/to/file.pdf');
$stream->serve(true, new WebFioriEmitter($response));
```

## Related Topics

* [Streaming Uploads](streaming-uploads.md) — Raw body uploads in constant memory
* [Resumable Uploads](resumable-uploads.md) — Chunked uploads with resume support
* [Web Services](web-services.md) — Create file upload APIs
* [Routing](routing.md) — Handle upload routes
* [The Class Response](class-response.md) — Send upload responses
