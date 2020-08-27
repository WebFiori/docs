
# The Class Response

<meta name="" description="">

In this page:

* [Introduction](#introduction)
* [Collecting Server Output](#collecting-server-output)
* [Sending Output](#sending-the-output)
* [Sending Custom Headers](#sending-custom-headers)

## Introduction

Normally, sending output to HTTP client in php is performed using [`echo`](https://www.php.net/manual/en/function.echo.php) or [`print`](https://www.php.net/manual/en/function.print.php). The framework has its own way for sending output to HTTP clients which is the class [`Response`](https://webfiori.com/docs/webfiori/entity/Response). The main aim of this class is to collect server output and send it back to the client after the server compeletly finished processing client request. For this reason, never use `echo`, `print` or `die` to show server output.

## Collecting Server Output

## Sending The Output

## Sending Custom Headers
