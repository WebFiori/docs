# Sending HTML Emails

<meta name="description" content="How to create HTML emails and send them.">

In this page:
* [Introduction](#introduction)
* [Configuration](#configuration)
* [Creating and Sending an Email](#creating-and-sending-an-email)
* [Attaching Files](#attaching-files)
* [SMTP Setup For Common Servers](#smtp-setup-for-common-servers)
  * [GMail SMTP Server](#gmail-smtp-server)
  * [Outlook SMTP Server](#outlook-smtp-server)

## Introduction
Email messages are considered as one of the most effective communication ways, and at some point, any website or web application will have to use them. WebFiori Framework has all needed tools to allow the application to be able to send HTML emails. Email messages are used in many ways. For example, they are used to activate user account, reset password, send news letters, etc...

> Note: This feature uses the library `webfiori\mail`. Please refer to the [documentation](https://github.com/WebFiori/mail) of the library for more information.

## Configuration

Before sending any email, first the developer have to add SMTP account information that will be used to send emails. The account can be added in two ways. Either by using the command `php webfiori add:smtp-connection` or by opening application configuration and adding the connection manually. The preferred way for adding SMTP connection information is the first one as it will validate connection information before storing them.

For every SMTP account, the following items must be specified:

* SMTP Server address.
* Server port (usually 25, 465 or 586).
* Username
* Password
* Sender address (usually same as username).
* Sender name.
* Account name.

The account name will act as an identifier for the account when sending a message.

## Creating and Sending an Email

Sending HTML email messages is performed using the class [`EmailMessage`](https://webfiori.com/docs/WebFiori/Framework/EmailMessage). This class has methods which are used to set the attributes of an email message such as its subject, importance and the people who will get the message. This class is usually used in the following way:

* Specify SMTP connection to use.
* Set the subject of the message.
* Specify the people who will receive the message.
* Write the message.
* Send the message.

There are other things which might be performed to the message before sending it. For example, the developer might specify the importance of the message or add attachments to it or add colors and styles. But for now, only the most basic use case is shown. The following PHP code shows how to create and send a basic email message.

``` php 
use WebFiori\Framework\EmailMessage;

//First thing to do, Specify SMTP account to use.
//In our example, the name was 'no-reply'.
$message = new EmailMessage('no-reply');
$message->setSubject('This is a Test Email');

//Adds a receiver. The method accepts a name and email address.
$message->addTo('user@example.com', 'Blog User');

//Insert a paragraph in the body of the message.
$p = $message->insert('p');

//Write some text
$p->text('This is a welcome message.');

//Final step, is sending the message.
$message->send();
```

The email will be sent in HTML format. To customize the content of the generated HTML, the developer must access the DOM of the message. Every email message instance is associated with an object of type [`HTMLDoc`](https://webfiori.com/docs/WebFiori/Ui/HTMLDoc). The document object can be accessed using the method [`EmailMessage::getDocument()`](https://webfiori.com/docs/WebFiori/Framework/EmailMessage#getDocument). In addition to that, it is possible to add objects of type [`HTMLNode`](https://webfiori.com/docs/WebFiori/Ui/HTMLNode) to the body of the message using the method [`EmailMessage::insert()`](https://webfiori.com/docs/WebFiori/Framework/EmailMessage#insert).

### Advanced Email Customization

Here are some examples of creating more sophisticated email messages:

``` php
use WebFiori\Framework\EmailMessage;

$message = new EmailMessage('no-reply');
$message->setSubject('Welcome to Our Platform!');

// Add multiple recipients
$message->addTo('user1@example.com', 'John Doe');
$message->addTo('user2@example.com', 'Jane Smith');

// Add CC and BCC recipients
$message->addCC('manager@example.com', 'Manager');
$message->addBCC('admin@example.com', 'Admin');

// Set message priority
$message->setPriority(1); // High priority

// Create a styled header
$header = $message->insert('div');
$header->setStyle([
    'background-color' => '#007bff',
    'color' => 'white',
    'padding' => '20px',
    'text-align' => 'center'
]);
$header->insert('h1')->text('Welcome to Our Platform!');

// Add main content
$content = $message->insert('div');
$content->setStyle(['padding' => '20px']);

$greeting = $content->insert('p');
$greeting->text('Dear User,');

$mainText = $content->insert('p');
$mainText->text('Thank you for joining our platform. We are excited to have you on board!');

// Add a button
$buttonContainer = $content->insert('div');
$buttonContainer->setStyle(['text-align' => 'center', 'margin' => '20px 0']);

$button = $buttonContainer->insert('a');
$button->setAttribute('href', 'https://example.com/activate');
$button->setStyle([
    'background-color' => '#28a745',
    'color' => 'white',
    'padding' => '10px 20px',
    'text-decoration' => 'none',
    'border-radius' => '5px',
    'display' => 'inline-block'
]);
$button->text('Activate Account');

// Add footer
$footer = $message->insert('div');
$footer->setStyle([
    'background-color' => '#f8f9fa',
    'padding' => '15px',
    'text-align' => 'center',
    'font-size' => '12px',
    'color' => '#666'
]);
$footer->text('© 2024 Your Company. All rights reserved.');

$message->send();
```

### Working with Email Templates

For consistent email design, you can create reusable email templates:

``` php
use WebFiori\Framework\EmailMessage;

class EmailTemplate {
    public static function createWelcomeEmail($smtpAccount, $recipientEmail, $recipientName, $activationLink) {
        $message = new EmailMessage($smtpAccount);
        $message->setSubject('Welcome! Please Activate Your Account');
        $message->addTo($recipientEmail, $recipientName);
        
        // Add template structure
        $container = $message->insert('div');
        $container->setStyle([
            'font-family' => 'Arial, sans-serif',
            'max-width' => '600px',
            'margin' => '0 auto'
        ]);
        
        // Header
        $header = $container->insert('div');
        $header->setStyle([
            'background' => 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
            'color' => 'white',
            'padding' => '30px',
            'text-align' => 'center'
        ]);
        $header->insert('h1')->text('Welcome to Our Platform!');
        
        // Content
        $content = $container->insert('div');
        $content->setStyle(['padding' => '30px']);
        
        $content->insert('p')->text("Hello $recipientName,");
        $content->insert('p')->text('Thank you for creating an account with us. To get started, please activate your account by clicking the button below:');
        
        // Activation button
        $buttonDiv = $content->insert('div');
        $buttonDiv->setStyle(['text-align' => 'center', 'margin' => '30px 0']);
        
        $button = $buttonDiv->insert('a');
        $button->setAttribute('href', $activationLink);
        $button->setStyle([
            'background-color' => '#4CAF50',
            'color' => 'white',
            'padding' => '15px 30px',
            'text-decoration' => 'none',
            'border-radius' => '5px',
            'display' => 'inline-block',
            'font-weight' => 'bold'
        ]);
        $button->text('Activate Account');
        
        $content->insert('p')->text('If you did not create this account, please ignore this email.');
        
        // Footer
        $footer = $container->insert('div');
        $footer->setStyle([
            'background-color' => '#f1f1f1',
            'padding' => '20px',
            'text-align' => 'center',
            'font-size' => '12px',
            'color' => '#666'
        ]);
        $footer->text('© 2024 Your Company Name. All rights reserved.');
        
        return $message;
    }
}

// Usage
$welcomeEmail = EmailTemplate::createWelcomeEmail(
    'no-reply',
    'newuser@example.com',
    'John Doe',
    'https://example.com/activate?token=abc123'
);
$welcomeEmail->send();
```

## Attaching Files

One of the features of the class [`EmailMessage`](https://webfiori.com/docs/WebFiori/Framework/EmailMessage) is the support for adding attachments to the message. In order to add attachments, the developer must use the class [`File`](https://webfiori.com/docs/WebFiori/File/File). This class is used to read and write files or send them back as a response. In order to add a file as an attachment, it must be first opened and then added to the message using the method [`EmailMessage::attach()`](https://webfiori.com/docs/WebFiori/Framework/EmailMessage#attach).

Assuming that we have a file which has the name "My CV.docx" in the root directory of the framework, The file can be attached to the email as follows:

``` php
use WebFiori\Framework\EmailMessage;
use WebFiori\File\File;

$message = new EmailMessage('no-reply');
$message->setSubject('Attached My Latest CV');

$message->addTo('user@example.com', 'Blog User');

$p = $message->insert('p');
$p->text('Attached is my newest CV. Please check it and reply to me if there is anything extra you need from me.');

//Open the file that will be added as an attachment.
$attachment = new File('My CV.docx', ROOT_DIR);
$message->attach($attachment);

$message->send();
```

### Multiple Attachments and File Types

You can attach multiple files of different types:

``` php
use WebFiori\Framework\EmailMessage;
use WebFiori\File\File;

$message = new EmailMessage('no-reply');
$message->setSubject('Monthly Report with Attachments');
$message->addTo('manager@example.com', 'Manager');

$message->insert('p')->text('Please find the monthly report and supporting documents attached.');

// Attach PDF report
$pdfReport = new File('monthly-report.pdf', ROOT_DIR . '/reports');
$message->attach($pdfReport);

// Attach Excel spreadsheet
$excelFile = new File('data-analysis.xlsx', ROOT_DIR . '/reports');
$message->attach($excelFile);

// Attach image
$chartImage = new File('sales-chart.png', ROOT_DIR . '/images');
$message->attach($chartImage);

$message->send();
```

### Error Handling and Validation

It's important to handle potential errors when sending emails:

``` php
use WebFiori\Framework\EmailMessage;
use WebFiori\File\File;

try {
    $message = new EmailMessage('no-reply');
    $message->setSubject('Test Email with Error Handling');
    $message->addTo('user@example.com', 'Test User');
    
    $message->insert('p')->text('This is a test email with proper error handling.');
    
    // Check if file exists before attaching
    $filePath = ROOT_DIR . '/documents/important.pdf';
    if (file_exists($filePath)) {
        $attachment = new File('important.pdf', ROOT_DIR . '/documents');
        $message->attach($attachment);
    } else {
        error_log("Attachment file not found: $filePath");
    }
    
    $result = $message->send();
    
    if ($result) {
        echo "Email sent successfully!";
    } else {
        echo "Failed to send email.";
    }
    
} catch (Exception $e) {
    error_log("Email sending failed: " . $e->getMessage());
    echo "An error occurred while sending the email.";
}
```

## SMTP Setup For Common Servers

This section shows configuration settings for two of the most commonly used SMTP servers and how to configure them.

### Gmail SMTP Server
* Server address: `smtp.gmail.com`
* Server port: `465` or `587`

When connecting to Gmail SMTP server, it is noticed that in some cases it fail even if credentials are correct. The error that the developer will get is similar to the following: `535-5.7.8 Username and Password not accepted. Learn more at 535 5.7.8  https://support.google.com/mail/?p=BadCredentials n15sm7406666wmq.38 - gsmtp`. To fix this issue, the developer must enable access for 'Less secure apps' in his account. More information can be found [here](https://support.google.com/accounts/answer/6010255?hl=en)


### Outlook SMTP Server
* Server address: `smtp.office365.com`
* Server port: `587`

In order to connect to Outlook SMTP, the developer must first generate an app password and use it as login password when adding the SMTP account. For more information on app passwords, check [here](https://support.microsoft.com/en-us/account-billing/using-app-passwords-with-apps-that-don-t-support-two-step-verification-5896ed9b-4263-e681-128a-a6f2979a7944).

## Related Articles

* [Command Line Interface](learn/command-line-interface) - Configure SMTP settings via CLI
* [Background Tasks](learn/background-tasks) - Send emails asynchronously
* [UI Package](learn/ui-package) - Create HTML email templates
* [Uploading Files](learn/uploading-files) - Attach uploaded files to emails
* [Web Services](learn/web-services) - Send emails through API endpoints
