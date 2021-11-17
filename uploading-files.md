# Uploading Files

In this page:
* [Introduction](#introduction)
* [The Class `Uploader`](#the-class-uploader)
* [Using the Class `Uploader`](#using-the-class-uploader)
  * [Uploading One File](#uploading-one-file)
  * [Uploading Multiple Files](#uploading-multiple-files)
* [Getting Uploaded File(s) Information](#getting-uploaded-files-information)

## Introduction

When trying to upload files in PHP, the developer have to access the global array [`$_FILES`](https://www.php.net/manual/en/reserved.variables.files.php) which can lead to a mess in some use cases. Also, it is difficult to use the global constant in case of multiple file uploads. More than that, there are security concerns when using the array directly for uploading user files.

The framework has a utility class that can be used to handle file uploads using mimimum amount of effort and code. The name of the class is [`Uploader`](https://webfiori.com/docs/webfiori/framework/Uploader) and its part of the namespace `webfiori\framework`.

## The Class `Uploader`

The class [`Uploader`](https://webfiori.com/docs/webfiori/framework/Uploader) is a utility class which is used to handle file uploads in PHP in a simple way. It has all necessary tools to handle one file upload or multiple files upload. The class will help in achieving the following:

* Restrict uploaded files types.
* Get MIME type of most file types using file extension only.
* Support for multiple uploads.
* View the status of each uploaded file in a simple way.
* No need to use the array `$_FILES` to upload files.
* Ability to create upload files API in matter of fiew lines
* Store the uploaded file(s) in a specific location on the server.
* Filtering for tainted user input.

## Using the Class `Uploader`

Usually, the most common use case is to upload a file to the server. This use case usually has the following steps:
* In the back-end:
  * Create an API for uploading files.
  * In the API, create new instance of the class [`Uploader`](https://webfiori.com/docs/webfiori/framework/Uploader#__construct).
  * Specify upload directory and allowed file types.
  * Specify the name of the input element that represents file upload input.
  * Call the method [`Uploader::upload()`](https://webfiori.com/docs/webfiori/framework/Uploader#upload) to initiate upload process.
  * Return back a JSON response that shows upload status of the file or files.
* In the front-end:
  * Create a new form.
  * Set the attribute `action` of the form to be the URL of the API.
  * Sets the attribute `method` of the form to `post`.
  * Sets the attribute `enctype` of the form to `multipart/form-data`
  * Add file input to the form and specify the attribute 'name' of the input.
  * Add submit input to the form.
  
  
### Uploading One File

Partial Backend Code (Shows Uploader initialization only):
``` php 
$uploader = new Uploader();

//sets upload path
$uploader->setUploadDir('\home\files\sys-uploads');

//allow pdf and word files
$uploader->addExts(['doc','docx','pdf']);

//sets the name of the location where files are kept in front-end
//This usually is the value of the attribute 'name' in case of HTML input element.
$uploader->setAssociatedFileName('files-input');

//start upload process. Replace if already uploaded.
$uploader->upload(true);

//shows a JSON string that shows upload status.
echo $uploader;
```

Assuming that a route to our upload API is created and the route is `apis/upload-files`, The front end code would be like the following:

``` php
use webfiori\framework\ui\WebPage;
use webfiori\ui\HTMLNode;
use webfiori\ui\Input;

class UploadView extends WebPage {
    __construct(){
        $form = $this->insert('form');
        //assume that upload API exist in the URL apis\upload-files
        $form->setAttributes([
            'method'=>'post'
            'action'=>'apis/upload-files'
            'enctype'=>'multipart/form-data'
        ]);
        $filesInput = new Input('file');
        $form->addChild($filesInput);
        
        //The attribute name must be set to the value which
        // is passed to the method Uploader::setAssociatedFileName()
        $file->setName('files-input')
        
        $submit = new Input('submit');
        $form->addChild($submit);
        
    }
}
return __NAMESPACE__
```

> Note: It is possible to skip setting the associated file name in server by adding a hidden input field with `name="file"` and `value` set to the name of the actual input which is used to upload files.

### Uploading Multiple Files

The code which is used to upload multiple files at one call is the same as the one which is used to upload one file. The diffrence is that we have to set the attribute 'multiple' of the file input element and set the name to a syntax which looks like an array as following:

``` php
$file->setAttribute('multiple')
$file->setName('files-input[]')
```

## Getting Uploaded File(s) Information

The method [`Uploader::upload()`](https://webfiori.com/docs/webfiori/framework/Uploader#upload) will return an associative array that contains upload status of uploaded files. In addition to getting files information as an associative array, it is possible to get uploaded files information as objects. The method [`Uploader::uploadAsFileObj()`](https://webfiori.com/docs/webfiori/framework/Uploader#uploadAsFileObj) will upload the files and return an array that contains objects of type [`UploadedFile`](https://webfiori.com/docs/webfiori/framework/UploadedFile). Also, it is possible to get uploaded files information even after the upload process is finished using the method [`Uploader::getFiles()`](https://webfiori.com/docs/webfiori/framework/Uploader#getFiles).


**Next: [Sending Emails](learn/sending-emails)**

**Previous: [Themes](learn/themes)**
