---
title: Learning about Microservice
layout: post
date: '2020-01-31 07:00:00'
tag:
- project
- microservice
- codeigniter
- php
image: assets/microservice-poc.png
headerImage: false
projects: true
hidden: true
description: Is microservice architecture the solution that fit all needs?
category: project
author: madeindra
externalLink: false
---

Earlier in 2020, I got a project assignment to do an example app for e-commerce application. I was tasked to code in PHP CodeIgniter3 & MariaDB as database.

The app was planned to have 3 specific class, those are Financial, Warehouse, and Order. The Warehouse class will handle item related functions, the Financial class will handle invoice and payment related function, and the Order class will handle the order from user.

A simple operation for ordering an item in app goes like this:

![Monolithic-Architecture](https://madeindra.github.io/assets/monolithic-poc.png)

**Start: User confirm an order**

1. Order class create the order on the database
2. Order class call Financial class to generate Invoice
3. Financial class create the invoice on the database
4. Financial class pass the invoice to Order class
5. Order class call Warehouse class to check item availability
6. Warehouse class fetch item's stock data in database
7. Warehouse class pass the item data to Order class

**End: User ready to pay the order**

After completing this step, I were tasked to decople the classes and move three classes to their own service.

By "service", it means the class will run on their own server instance and their own database instance, thus making them unable to call each other like in the previous model.

This raise a questions, how could they communicate?

One solution is to call the function using API, but this is not desired, because it will cause the service to go through external mean of communication. The idea of decopling was to let the services communicate to each other without using external mean of communication.

**Enter Microservice**

At this point, I reached an answer, it was to add another dependency that will pass a message to each service. 

The choice was to use either RabbitMQ or Kafka, and on this project I choose RabbitMQ.

By using RabbitMQ AMQP communication protocol, each service will pass a service to RabbitMQ, and RabbitMQ will send the message to the designated service. After the service recived the message,the service will run a function according to the message, and create another message to the service that send the message before.

Ordering an item in the app goes like this:

![Microservice-Architecture](https://madeindra.github.io/assets/microservice-poc.png)

**Start: User confirm an order**

1: Order Service fetches the Order Detail of (orderId 1), Order Service get (productId 1, quantity 10) detail.

2: Order Service pass (orderId 1, productId 1, quantity 10) to Order Worker to create invoice and to check if the product is in stock.

3: Order Worker sends a message (orderId 1, productId 1, quantity 10) to Exchange 1.

4a, 4b: Exchange 1 received message (orderId 1, productId 1, quantity 10), as it is a Direct Exchange, it forwards the message to Subscribing Queue(s).

5a: Financial Worker received a message (orderId 1, productId 1, quantity 10), then Financial Worker passes the message to Financial Service to be processed.

5b: Warehouse Worker received a message (orderId 1, productId 1, quantity 10), then Warehouse Worker pass only (productId 1, quantity 10) to Warehouse Service to be checked.

6a: Financial Service creates an Invoice based on the message (orderId 1, productId 1, quantity 10).

6b: Warehouse Service prepares a query to check stock based on the productId 1.

7a: Financial Service successfully created the Invoice on Financial Database.

7b: Warehouse Service fetches stock from Warehouse Database.

8a: Financial Service tells Financial Worker that it has created the Invoice.

8b: Warehouse Service gets information (stock 50) and compares it to quantity 10, because (stock > quantity), the order can be processed, send (productId 1, quantity 10, inStock true) to Warehouse Worker.

9: Warehouse Worker sends a message (productId 1, quantity 10, inStock true) to Financial Queue.

10: Financial Worker receives a message (productId 1, quantity 10, inStock true) stored in Financial Queue.

11: Financial Worker then passes the message to Financial Service to be processed.

12: Financial Service updates the Invoice status.

13: Financial Service tells Financial Worker that it has updated the Invoice.

14: Financial Worker sends a message (productId 1, quantity 10, invoice true) to Exchange 2.

15a, 15b: Exchange 2 received a message (productId 1, quantity 10, invoice true), as it is a Direct Exchange, it forwards the message to Subscribing Queue(s).

16a: Order Worker received a message (productId 1, quantity 10, invoice true), then Order Worker passes the message to Order Service to be processed.

16b: Warehouse Worker received a message (productId 1, quantity 10, invoice true), then Warehouse Worker passes the message to Warehouse Service to be processed.

17a: Order Worker completes the user's check-out process.

17b: Warehouse Service prepares a query to decrease the product's stock based on the message (productId 1, quantity 10, invoice true).

18: Warehouse Service decreases the stock of the product in the Warehouse Database.

19: Warehouse Service tells Warehouse Worker that it has updated the Stock.

**End: User ready to pay the order**

It went really complicated for a simple operation, right?

But this approach make it easier to maintain a single service, a different team can work on a different service at the same time. But in my experience, this made it harder to debug.

Oh right, you must notice the **worker** on the picture, this worker is made by creating a single `.php` file that run by using a terminal.

**Why was worker needed?**

For this approach, the PHP code need to process the user command and send a message to RabbitMQ, but if the code wait for RabbitMQ message sent before continuing the process, it could take a long time, thus it need to be run asyncronously.

At the time, I was unable to find another approach than using worked because PHP can't run a code asyncronously. So, the worker need to run separately to the app, to do this, I run it using terminal command.

This was how I learn about microservice, for those who interested to follow my approach, you can take a look at <a href="https://github.com/madeindra/codeigniter-microservice">Microservice in CodeIgniter 3 (proof-of-concept)</a> on my Github .

This is by no mean the best approach to microservice, there's a lot to improve and there might be a wrong step to do microservice.
