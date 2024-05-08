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

Before sending any email, first the developer have to add SMTP account information that will be used to send emails. The account can be added in two ways. Either by using the command `php webfiori add` or by opening application configuration and adding the connection manually. The prefered way for adding SMTP connection information is the first one as it will validate connection information before storing them.

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

Sending HTML email messages is performed using the class [`EmailMessage`](https://webfiori.com/docs/webfiori/framework/mail/EmailMessage). This class has methods which are used to set the attributes of an email message such as its subject, importance and the people who will get the message. This class is usually used in the following way:

* Specify SMTP connection to use.
* Set the subject of the message.
* Specify the people who will receive the message.
* Write the message.
* Send the message.

There are other things which might be performed to the message before sending it. For example, the developer might specify the importance of the message or add attachments to it or add colors and styles. But for now, only the most basic use case is shown. The following PHP code shows how to create and send a basic email message.

``` php 
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

The email will be sent in HTML format. To customize the content of the generated HTML, the developer must access the DOM of the message. Every email message instance is associated with an object of type [`HTMLDoc`](https://webfiori.com/docs//webfiori/ui/HTMLDoc). The document object can be accessed using the method [`EmailMessage::getDocument()`](https://webfiori.com/docs/webfiori/framework/mail/EmailMessage#getDocument). In addition to that, it is possible to add objects of type [`HTMLNode`](https://webfiori.com/docs/webfiori/ui/HTMLNode) to the body of the message using the method [`EmailMessage::insert()`](https://webfiori.com/docs/webfiori/framework/mail/EmailMessage#insert).

## Attaching Files

One of the features of the class [`EmailMessage`](https://webfiori.com/docs/webfiori/framework/mail/EmailMessage) is the support for adding attachments to the message. In order to add attachments, the developer must use the class [`File`](https://webfiori.com/docs/webfiori/file/File). This class is used to read and write files or send them back as a response. In order to add a file as an attachment, it must be first opened and then added to the message using the method [`EmailMessage::attach()`](https://webfiori.com/docs/webfiori/framework/mail/EmailMessage#attach).

Assuming that we have a file which has the name "My CV.docx" in the root directory of the framework, The file can be attached to the email as follows:
``` php
$message = new EmailMessage('no-replay');
$message->setSubject('Attached My Latest CV');

$message->addTo('user@example.com', 'Blog User')

$p = $message->insert('p');
$p->text('Attached is my newest CV. Please check it and replay to me if there is any thing extra you need from me.')

//Open the file that will be added as an attachment.
$attachment = new File('My CV.docx',ROOT_DIR);
$message->attach($attachment);

$message->send()
```

## SMTP Setup For Common Servers

This section shows configuration settings for two of the most commonly used SMTP servers and how to configure them.

### Gmail SMTP Server
* Server address: `smtp.gmail.com`
* Server port: `465` or `587`

When connecting to Gmail SMTP server, it is noticed that in some cases it fail even if credentials are correct. The error that the developer will get is similiar to the following: `535-5.7.8 Username and Password not accepted. Learn more at 535 5.7.8  https://support.google.com/mail/?p=BadCredentials n15sm7406666wmq.38 - gsmtp`. To fix this issue, the developer must enable access for 'Less secure apps' in his account. More information can be found [here](https://support.google.com/accounts/answer/6010255?hl=en)


### Outlook SMTP Server
* Server address: `smtp.office365.com`
* Server port: `587`

In order to connect to Outlook SMTP, the developer must first generate an app password and use it as login password when adding the SMTP account. For more information on app passwords, check [here](https://support.microsoft.com/en-us/account-billing/using-app-passwords-with-apps-that-don-t-support-two-step-verification-5896ed9b-4263-e681-128a-a6f2979a7944).

**Next: [Command Line Interface](learn/command-line-interface)**

**Previous: [Uploading Files](learn/uploading-files)**
