---
title: "A Step Into Messaging Systems"
date: 2018-11-05T05:17:17+02:00
draft: false
---

Around two years ago, I had to work on a school project that I was supposed to
build a GUI application using SOAP services, Apache ActiveMQ. The hardest part
was to use ActiveMQ somewhow, finally after many failings I got it working. The
funny part was though that I did not know what exactly I was doing. Since then I
could here about messaging systems here and there. After some thinking, I
believe I found a simple use case that would provide me the playground to try
new things.

The use case is that whenever a new user registers for the application, they
should receive an email with their verification/activation link. User does not
have wait for registration response while system tries to send the email. In
nature, it can be a asynchronous operation. Even in case of a failure, user may
ask the system to resend the email. It is really simply that it could be tried
out without affecting other parts of the system. And anyway, Todarch does not
have this functionality yet.

The setup is as follows: um service created a new event on user registration.
This event object would have the email, link and other piece of information if
needed. Somehow this event is going to get published on the messaging system.
Somewhere else another service called cm is going to listen to some
channel/queue/topic/exchange for this kinds of events, and act on it whenever it
gets a new one.

### Kafka

The first try was with Apache Kafka. We do not have enough resources or
experience to maintain our own Kafka installation.
[Cloudkarafka](https://www.cloudkarafka.com/) provides Kafka as a service. They
have some examples to help you use it in your application easily.

It was not to complicated to start with. Firstly, I have used 'spring-kafka'.
The consumer side code is [over
here](https://github.com/todarch/todarch-cm/commit/25ef9ca0c4550366c37dfcff914d1e42d8a89311)
and [the
configuration](https://github.com/todarch/todarch-config/commit/a3f7576dc4049e9dba9241a22a9e98fffa28f71a#diff-4616c5b42ab6380188d63e2b66350333).
It was some time ago but I remember that I struggled the serialization of the
message. Afterall, I managed to get it working in the way that I designed in my
head. However, I was not confortable with the result because of a few reasons:

- Free Kafka service is good for trying out new things, but in a long run it
  would to invest on that side. Spending money is not an option now.

- Maintaining our own Kafka was not an option either. Lack of resources and experience.

- On the code level with serialization and configuration of Kafka was not easy
  and comfortable for a starter.

So I put the idea aside for some time. Until I started to play with Spring Cloud
Contract. Creating REST contracts between provider and consumer was expected is
expected. Even manually and with some mistake I could write those tests myself.
The interesting part for me to be able do this for messaging systems. I needed
to take "email sending idea" off the shelfe. I might write another post on
Spring Cloud Contract.

## Down the Rabbit Hole

While watching [this contract demo
video](https://www.youtube.com/watch?v=MDydAqL4mYE), and [Spring Cloud Stream
Tips by Josh Long](https://www.youtube.com/watch?v=HQ00E60kB6c) I realized that directly
configuring Kafka or any other messaging system is not the Springy way. I have
got on searching and found another great project 'Spring Cloud Stream'. It
abstracts over messaging system, does the configuration work for you. The only
thing left for you to write code for your business and throw in messaging system
dependency. For now I see that they have it for Rabbitmq and Kafka. The promise
of the project is that you could change your mind on which messaging system you
would like to use but would not have to change your code. Because of
the reasons mentioned above, I decided to try out RabbitMQ. Actually it was
easier than I expected, just run it as Docker container, did not require lots of
resources. The management UI helped a little to see what is going on within
RabbitMQ itself.

I achieved to configure successfully to publish the message and receive it on the other
side. I know the configuration is done. Easily. Now I could go back to defining
my contracts. That worked well too. I learned new parts of those projects. When
things all worked together. I had two separated services with only sharing a
destination string. With the contract tests in place, I was sure both sides were
talking in the same language. 


## Way to Production

Even the activation email functionality is not going to reflected on the UI yet.
Why not test what was already done on production. Firstly, an instace of
RabbitMQ had to configured. That was easily using docker-compose files. I did
not even bothered to customize the instance, going to get into that when the
need arises. It may not be the best idea to deploy RabbitMQ with management ui
if you already know ins and outs of it. While having Traefik to configure all
why not.

```yml
mq:
  image: rabbitmq:3-management
  ports:
    - "5672:5672"
    - "15672:15672"
  expose:
    - "5672"
    - "15672"
  environment: # override this values in docker-compose.prod.yml
    - RABBITMQ_DEFAULT_USER=guest
    - RABBITMQ_DEFAULT_PASS=guest
  restart: always
```

The other part of the deployment was cm service. Communication service does not
have dependent yet, so why not configure it for the fast deployment. More things
were on place with CircleCI, deploy.sh script, jib plugin. A minute after
merging into master it was up and running. Of course not. There were missing
properties etc. That was the reason that we deployed the when is not needed in
the first placee right?
