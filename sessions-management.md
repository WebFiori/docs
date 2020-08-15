
# Sessions Management

<meta name="description" content="Every web application must have a way to manage users sessions. The management may differe from one web development framework to another. A session in simple terms is a way to keep track of user intractions through your web application.">

In this page:
* [Introduction](#introduction)

## Introduction

Every web application must have a way to manage users sessions. The management may differe from one web development framework to another. A session in simple terms is a way to keep track of user intractions through your web application. They can be used to keep state between different requests for the same user. WebFiori framework provides one class at which the developer can use to manage application sessions. The name of the class is [`SessionsManager`](https://webfiori.com/docs/webfiori/entity/session/SessionsManager). Sessions in WebFiori framework can be used with and without HTTP as long as the client keep session ID in his side.

## Starting New Session

Starting new session is very simple. The developer can use the method [`SessionsManager::start()`](https://webfiori.com/docs/webfiori/entity/session/SessionsManager#start). This method accepts two parameters. The first one is session name and the second one is an associative array of options. Calling this method will create new session which is persistent. The duration of the session will be 2 hours.

``` php
SessionsManager::start('hello-session');
echo SessionsManager::getActiveSession();
```

The output of the code would be something similar to this:

``` json
{
    "name": "hello-session",
    "started-at": 1597509807,
    "duration": 7200,
    "resumed-at": 1597509807,
    "passed-time": 0,
    "remaining-time": 7200,
    "language": "EN",
    "id": "d01de401aa3d7f29eb24f6eb74a5225d473f2e1c54b9f4cb6d9fc9cc3de87f41",
    "is-refresh": false,
    "is-persistent": true,
    "status": "status_new",
    "user": {
        "user-id": -1,
        "email": "",
        "display-name": null,
        "username": ""
    },
    "vars": []
}
```
### Customizing The Session

It is possible to customize the session and set the following properties during initialization:
* Duration of the sesstion.
* Is the session will be refreshed with every request or not.
* Make the session non-persistent.

The following code shows how to create create a session with specific duration and it's remaining time refreshes with every request.

``` php 
SessionsManager::start('hello-session', [
    'duration' => 30, // 30 Minutes
    'refresh' => true
]);
```

## Resuming a Session

To resume a session, simply use the same method which is used to start a new session.
``` php
SessionsManager::start('hello-session');
SessionsManager::close();
SessionsManager::start('hello-session');
echo SessionsManager::getActiveSession();
```

## Destroying a Session
## Adding Data to a Session
## Multiple Sessions
