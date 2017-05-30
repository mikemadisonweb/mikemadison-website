---
title: Using RabbitMQ in your Yii2 applications
date: 2017-05-27
featured_image: using-rabbitmq-in-your-yii2-applications.jpeg
tags:
- PHP
- Yii2
- RabbitMQ
---
This article describes an easy way to get started with RabbitMQ if you are using PHP and Yii2 Framework. We will try to do the job using yii2-rabbitmq extension, for that purpose we will build a simple messaging app.

# Architecture
RabbitMQ is an open source software written in Erlang, developed and maintained by Pivotal. Despite the fact that it was originally designed for AMQP, it support multiple protocols like STOMP, MQTT, HTTP. We will focus on default AMQP implementation. For the sake of tutorial we will build photo searching app built on top of Flickr API. As we want it to be modern, reactive and basically cool, we will send our request asynchronously and present the results without page refresh using Websockets. Lets assume we have that PHP part, which are playing a central role as it makes API requests and download found images and NodeJs part that you use to establish websocket connection with client and send images through it.
img: application schema
But before we get our hand dirty, let's learn some basic terms and structures of RabbitMQ.