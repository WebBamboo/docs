---
author:
  name: Linode Community
  email: docs@linode.com
description: How to implement fast and reliable real-time communication in your PHP/Javascript applications using Mercure.
keywords: ['mercure','javascript','php']
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
published: 2020-02-14
modified_by:
  name: Linode
image: mercurelogo.png
title: How to install Mercure on Debian - A Tutorial
h1_title: How to install Mercure on Debian
contributor:
  name: Pavel Petrov
  link: https://github.com/WebBamboo/
external_resources:
- '[Mercure](https://mercure.rocks/)'
- '[JWT](https://jwt.io)'
- '[WebSockets API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)'
- '[Server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)'
- '[Polling](https://en.wikipedia.org/wiki/Polling_(computer_science))'
audiences: ["beginner"]
languages: ["javascript", "php", "bash"]
---

## Introduction

Mercure is an open source tool built on top of HTTP and SSE which allows you to easily implement true real-time communication in your web applications. While the idea of this is not unique or new, Mercure lets you implement it fairly easily. I’m using the term “True real-time communication” because developers frequently mock real time communication by polling. Polling is great for small scale applications, but for large scale real-time communication can save you a whole bunch of useless requests thus traffic and CPU load. But what about WebSocket API isn’t that exactly the same? Well it is and it isn’t. WebSocket API is low level, whereas Mercure is High Level. Using Mercure you don’t have to manually implement authorization, re-connection, presence API etc.
In a nutshell if you need a persistent connection between a client and a server which can be used by both to send and receive data and you don’t want to implement it all from scratch using WebSockets - Mercure is the tool for the job.

In this guide, you will learn:

- [How to install Mercure](#how-to-install-mercure) 
- [How to configure Mercure as a Service](#how-to-configure-mercure-as-a-service) on Ubuntu/Debian/Linux Mint
- [Using Mercure in an example](#using-mercure-in-an-example) to verify it works

{{< note >}}
You can [download all of the example files for this guide here](how-to-install-mercure.tar.gz).
{{< /note >}}

## How to install Mercure
First thing is first, head to the [Releases page in the Mercure GitHub](https://github.com/dunglas/mercure/releases). In this guide I’ll be using the v0.10.2 for Linux x86/64 (mercure_0.10.2_Linux_x86_64.tar.gz).

```bash
wget https://github.com/dunglas/mercure/releases/download/v0.10.2/mercure_0.10.2_Linux_x86_64.tar.gz && mkdir mercure && tar -zxvf mercure_0.10.2_Linux_x86_64.tar.gz -C mercure
```

Now, the Mercure executable is extracted and you can enter the directory and run it:

        cd mercure && ./mercure

Which should output: 
{{< output >}}
Invalid config: one of “jwt_key” or “publisher_jwt_key” configuration parameter must be defined. 
{{< /output >}}
But what is JWT? JWT or JSON Web Token is an open standard for securely transmitting information between parties in JSON. I won’t go into too much detail about JWT if you’re interested head to their homepage to learn more.
Let’s supply a jwt-key and rerun mercure:

        ./mercure --jwt-key=’!ChangeMe!’ --addr=’localhost:3000’ --allow-anonymous --cors-allowed-origins=’*’

Now everything should be running and you should be able to open the “http://127.0.0.1:3000” address in your browser and see:

{{< output >}}
Welcome to Mercure!
{{< /output >}}

## How to configure Mercure as a Service

So far so good, but there is one problem: if you terminate your terminal or the command gets closed or fails your mercure server will close and your customers will lose their persistent connections and live updates.
In order to avoid accidental closing of our Mercure Hub I will use a software called Supervisor or Supervisor daemon(Supervisord). In a few words Supervisord is a process management system that will keep our process running, reboot on crashes and enable logging.

### Installing supervisor
As the root user run the following command to install the Supervisor package:
```bash
apt-get install supervisor
```
{{< note >}}
You can ensure the supervisor is running with the following command:
```bash
systemctl status supervisor
```
![Supervisord - Status](supervisord-status.png "Supervisord - Status")
{{< /note >}}

### Setting up supervisor for Mercure

The configuration files for all programs supervisor is managing can be found in /etc/supervisor/conf.d so in order to add mercure we’ll create a new file there:

{{< file "/etc/supervisor/conf.d/mercure.conf" bash >}}
[program:mercure]
command=/path/to/mercure
process_name=%(program_name)s_%(process_num)s
numprocs=1
environment=JWT_KEY="!ChangeMe!",ADDR=':3000', DEMO=1, ALLOW_ANONYMOUS=1, CORS_ALLOWED_ORIGINS=*, PUBLISH_ALLOWED_ORIGINS='*', USE_FORWARDED_HEADERS=1, DEBUG=1
directory=/tmp
autostart=true
autorestart=true
startsecs=5
startretries=10
user=www-data
redirect_stderr=false
stdout_capture_maxbytes=1MB
stderr_capture_maxbytes=1MB
stdout_logfile=/path/to/mercure/out.log
stderr_logfile=/path/to/mercure/error.log
{{< /file >}}

It is very important to change the path in the file to the path of mercure in your system, and the JWT_KEY to the key you’ll be using. This configures Supervisor to provide the needed environment variables which Mercure uses for options, which you can read more about in the [config docs of Mercure here](https://mercure.rocks/docs/hub/config). Most importantly our script will autostart, autorestart and log to the /path/to/mercure/out.log and /path/to/mercure/error.log files.

Now we have to make supervisor aware of our new config file by running the following command:

        supervisorctl reread

The output to which should be:

{{< output >}}
mercure: available
{{< /output >}}

Now in order to run the loaded config file we run:

        supervisorctl update

And to make sure everything is running correctly you should run:

        supervisorctl status

To which the output should be similar to:
{{< output >}}
Mercure:mercure_0        RUNNING     pid XXX, uptime 0:01:00
{{< /output >}}

Now that we’ve installed mercure and made sure the server will autostart and autoreload on crashes we can test the whole thing with a simple example.

## Using Mercure in an example

### Generate a JWT Token
In order for our Publisher to be authorized to push updates to our Mercure Hub we need to generate a JWT token. 
For our example we want our publisher to be able to push to all topics so our payload should contain at least the following structure:
```json
{
  "mercure": {
    "publish": [
      "*"
    ]
  }
}
```

We can easily generate a token with this payload in the [JWT.io](https://jwt.io/#debugger-io?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXJjdXJlIjp7InB1Ymxpc2giOlsiKiJdfX0.obDjwCgqtPuIvwBlTxUEmibbBf0zypKCNzNKP7Op2UM) website, changing the JWT secret key with the one we configured above. In this case “!ChangeMe!”

![JWT Token Generation](jwt-token.png "JWT Token Generation")
And the configured JWT Token is:
```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXJjdXJlIjp7InB1Ymxpc2giOlsiKiJdfX0.obDjwCgqtPuIvwBlTxUEmibbBf0zypKCNzNKP7Op2UM
```

### Sending an update to a Mercure topic using PHP
The easiest way I've found to push updates to Mercure using PHP is by using the open-source [symfony/mercure package](https://github.com/symfony/mercure):

`composer require symfony/mercure`

Then in order to push an update to the Mercure Hub you can use the packagein the following way:
{{< file "Pusher.php" php >}}
<?php
require_once("vendor/autoload.php");
 
// change these values accordingly to your hub installation
define('HUB_URL', 'http://127.0.0.1:3000/.well-known/mercure');
define('JWT', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXJjdXJlIjp7InB1Ymxpc2giOlsiKiJdfX0.obDjwCgqtPuIvwBlTxUEmibbBf0zypKCNzNKP7Op2UM');
 
use Symfony\Component\Mercure\Jwt\StaticJwtProvider;
use Symfony\Component\Mercure\Publisher;
use Symfony\Component\Mercure\Update;
 
$publisher = new Publisher(HUB_URL, new StaticJwtProvider(JWT));
// Serialize the update, and dispatch it to the hub, that will broadcast it to the clients
$id = $publisher(new Update('https://example.com/linode/1.jsonld', 'Hi from Linode!'));

{{< /file >}}
The idea of the pusher is to tell the Mercure Hub to push an update to all clients subscribed to “https://example.com/linode/1.jsonld” with the payload “Hi from Linode!”

### Subscribing to a topic in Mercure using Javascript

Now in order to properly test our system we need a client. Lets client an index.html file with the following contents:
{{< file "index.html" html >}}
<script>
   const url = new URL('http://localhost:3000/.well-known/mercure');
   url.searchParams.append('topic', 'https://example.com/linode/{id}');
   // The URL class is a convenient way to generate URLs such as http://localhost:3000/.well-known/mercure?topic=https://example.com/books/{id}
 
   const eventSource = new EventSource(url);
 
   // The callback will be called every time an update is published
   eventSource.onmessage = e => alert(e.data); // do something with the payload
</script>
{{< /file >}}
This subscribes our browser to the topic  https://example.com/books/{id} where {id} is a parameter. Now test it out by openning index.html and running the Pusher.php by:

        php Pusher.php


Your browser should have opened a JavaScript alert instantly.

![Success](success.png "Success")

Now test opening the index.html a few more times and run the Pusher again. The tabs in this case are somewhat similar to multiple clients and running the pusher sends real-time updates to all tabs.