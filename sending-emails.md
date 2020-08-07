# Sending HTML Emails

In this page:
* [Introduction](#introduction)
* [Required Classes for Sending an Email](#required-classes-for-sending-an-email)
  * [The Class `SMTPAccount`](#the-class-smtpaccount)
  * [The Class `MailConfig`](#the-class-mailconfig)
  * [The Class `EmailMessage`](#the-class-emailmessage)
  * [The Class `EmailController`](#the-class-emailcontroller)
* [Sending an Email](#sending-an-email)
  * [Storing SMTP Connection](#storing-smtp-connection)
  * [Creating and Sending The Message](#creating-and-sending-the-message)
* [Attaching Files](#attaching-files)

## Introduction
One of the important features of any web application is the ability to send email messages. WebFiori Framework has all needed tools to allow the system be able to send HTML emails. Email messages are used in many ways. For example, they are used to activate user account, reset password, send ads, etc...

## Required Classes for Sending an Email
In order to send emails using the framework you need to learn at least about 3 classes. These classes are:
* [`SMTPAccount`](https://webfiori.com/docs/webfiori/entity/mail/SMTPAccount)
* [`MailConfig`](https://webfiori.com/docs/webfiori/config/MailConfig)
* [`EmailMessage`](https://webfiori.com/docs/webfiori/entity/mail/EmailMessage)

In addition to the given 3, there exist one extra class which is [`EmailController`](https://webfiori.com/docs/webfiori/logic/EmailController). This one acts as a utility class.

### The Class `SMTPAccount`

The class [`SMTPAccount`](https://webfiori.com/docs/webfiori/entity/mail/SMTPAccount) represents SMTP account connection information. It is used to keep SMTP server address, port number, SMTP credentials in addition to the email address and sender name. The first thing that a developer must have in order to send emails is SMTP account that can be used to connect with mailing server.

### The Class `MailConfig`

The class [`MailConfig`](https://webfiori.com/docs/webfiori/config/MailConfig) is a configuration class which is used to store SMTP connections which are used by the system to send emails. The class can be used to store unlimited number of SMTP accounts. Every account is kept as an object of type [`SMTPAccount`](https://webfiori.com/docs/webfiori/entity/mail/SMTPAccount).

### The Class `EmailMessage`

The class [`EmailMessage`](https://webfiori.com/docs/webfiori/entity/mail/EmailMessage) is used to send the actual email. It has static methods which are used to set many attributes of an email message such as the subject, the people who will receive the email and its attachments.

### The Class `EmailController`

The class [`EmailController`](https://webfiori.com/docs/webfiori/logic/EmailController) is a utility class which acts as an interface between the email message and the logic which is used to send the message. In addition, this class is used to modify the class The class [`MailConfig`](https://webfiori.com/docs/webfiori/config/MailConfig) programatically by adding and removing connections.

## Sending an Email

In order to send an email, first we must have SMTP account that can connect to SMTP server. Once we have that, the following steps must be performed in order to send an email message:

* Store SMTP connection information.
* Initialize your message using the method [`EmailMessage::createInstance()`](https://webfiori.com/docs/webfiori/entity/mail/EmailMessage#createInstance).
* Specify the subject of the message.
* Specify the people who will get the message (Original copy, CC or BCC).
* Add content to the message body and add attachments (if any).
* Send the message by ccalling the method EmailMessage::send()

The first step is usually performed once. After storing connection information, the connection can be used multiple times for sending different messages.

### Storing SMTP Connection

As we have said before, the class [`MailConfig`](https://webfiori.com/docs/webfiori/config/MailConfig) is used to store SMTP connections. There are two ways to add new SMTP connections to the class. It is possible to open the file which has the class code and add the connection manually, or we can add the connection using command line interface. The second way is recommended as it will validate connection information before storing them. For the time being, we will use the manual way.

Using the manual way, we add connection information by modifying the code in the constructor of the class "MailConfig". Lets assume that we have SMTP account that has the following info:
* Server Address: "mail.example.com".
* Port: "465".
* Username: "no-replay@example.com".
* Password: "123654".
* Sender Name: "System Notifier".
* Sender Address: "no-replay@example.com".

Using the given info, Connection information can be added maually as follows:

``` php
//Inside the class MailConfig's constructor...
$account00 = new SMTPAccount();
$account00->setServerAddress('mail.example.com')
$account00->setPort(465);
$account00->setUsername('no-replay@example.com');
$account00->setPassword('123654');

//The next two can be set to any value.
//They are the name and sender address which appear in email clients.
$account00->setName('System Notifier');
$account00->setAddress('no-replay@example.com');

//Finally, add the account.
//We must give our account a name in order to use it later.
$this->addSMTPAccount('no-replay-smtp',$account00);
```

### Creating and Sending The Message

Sending HTML email messages is performed using the class [`EmailMessage::createInstance()`](https://webfiori.com/docs/webfiori/entity/mail/EmailMessage#createInstance). This class has only static methods which are used to set the attributes of an email message such as its subject, importance and the people who will get the message. This class is usually used in the following way:

* Specify SMTP connection to use.
* Set the subject of the message.
* Specify the people who will receive the message.
* Write the email message.
* Send the message.

There are other things which might be performed to the message before sending it. For example, we might specify the importance of the message or add attachments to it or add colorsand styles. But for now, we will look at the most basic use case. The following PHP code shows how to create and send a basic email message.

``` php 
//First thing to do, Specify SMTP account to use.
//In our example, the name was 'no-replay-smtp'.
EmailMessage::createInstance('no-replay-smtp');
EmailMessage::subject('This is a Test Email');

//Adds a receiver. The method accepts a name and email address.
EmailMessage::addReceiver('Blog User','user@example.com');

//Write some text to the body of the message.
EmailMessage::write('This is a welcome message.');

//Final step, is sending the message.
EmailMessage::send();
```

The email will be sent in HTML format. To customize the content of the generated HTML, the developer must access the DOM of the message. Every instance which is created using the class has an object of type [`HTMLDoc`](https://webfiori.com/docs/phpStructs/html/HTMLDoc) which is associated with it. The document object can be accessed using the method [`EmailMessage::documen()`](https://webfiori.com/docs/webfiori/entity/mail/EmailMessage#documen). In addition to that, it is possible to add objects of type [`HTMLNode`](https://webfiori.com/docs/phpStructs/html/HTMLNode) to the body of the message using the method [`EmailMessage::insertNode()`](https://webfiori.com/docs/webfiori/entity/mail/EmailMessage#insertNode).

## Attaching Files

One of the features of the class [`EmailMessage`](https://webfiori.com/docs/webfiori/entity/mail/EmailMessage) is the support for adding attachments to the message. In order to add attachments to the message, the developer must use the class [`File`](https://webfiori.com/docs/webfiori/entity/File). This class is used to read and write files or send them back as a response. In order to add a file as an attachment, it must be first opened and then added to the message using the method [`EmailMessage::attach()`](https://webfiori.com/docs/webfiori/entity/mail/EmailMessage#attach).

Assuming that we have a file which has the name "My CV.docx" in the root directory of the framework, The file can be attached to the email as follows:
``` php
EmailMessage::createInstance('no-replay-smtp');
EmailMessage::subject('Attached My Latest CV');

EmailMessage::addReceiver('Blog User','user@example.com')

//Write some text to the body of the message.
EmailMessage::write('Attached is my newest CV. Please check it and replay to me if there is any thing extra you need from me.')

//Open the file that will be added as an attachment.
$attachment = new File('My CV.docx',ROOT_DIR);
//We have to read the file before adding it as an attachment.
$attachment->read();
//Finally, we add it to our message.
EmailMessage::attach($attachment);

EmailMessage::send()
```
