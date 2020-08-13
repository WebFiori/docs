
# Sessions Management

<meta name="description" content=""

In this page:
[Introduction](#introduction)

## Introduction

Every web application must have a way to manage users sessions. The management may differe from one web development framework to another. A session in simple terms is a way to keep track of user intractions through your web application. They can be used to keep state between different requests for the same user. WebFiori framework provides one class at which the developer can use to manage application sessions. The name of the class is [`SessionsManager`](https://webfiori.com/docs/webfiori/entity/session/SessionsManager). Sessions in WebFiori framework can be used with and without HTTP as long as the client keep session ID in his side.

## Starting New Session

Starting new session is very simple. The developer can use the method [`SessionsManager::start()`](https://webfiori.com/docs/webfiori/entity/session/SessionsManager#start). This method accepts two parameters. The first one is session name and the second one is an associative array of options.

``` php
SessionsManager::start('hello-session');
echo SessionsManager::getActiveSession();
```

## Resuming a Session
## Destroying a Session
## Adding Data to a Session
## Multiple Sessions
