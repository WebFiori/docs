# The Labriry JsonX

<meta name="description" content="The library JsonX can be used to create well formatted JSON strings. This page explains how it can be used.">

In this page:

* [Introduction](#introduction)

## Introduction

When desining web services, the developer must deside on the format at which the server will send back to the client after processing the request. One of the most widely used format is called [JSON](https://www.json.org/json-en.html). PHP does provide functions for encoding and decoding JSON using the methods [`json_encode()`](https://www.php.net/manual/en/function.json-encode.php). And [`json_decode()`](https://www.php.net/manual/en/function.json-decode.php). Since the framework promots the use of OOP approach, it uses a library called [`JsonX`](https://github.com/usernane/jsonx) to create well formatted JSON output.

## Library Structure

The lirary has only one class and one interface. The class [`JsonX`](https://webfiori.com/docs/jsonx/JsonX) is the core of the library. This class is the one which is used to create the final JSON output. In addition to that, it can be used to parse JSON strings and convert them to `JsonX` objects. 

The interface [`JsonI`](https://webfiori.com/docs/jsonx/JsonI) can be used to customize JSON representation of objects which implement it when encoded. The interface has only one method which is [`JsonI::toJSON()`](https://webfiori.com/docs/jsonx/JsonI#toJSON).
