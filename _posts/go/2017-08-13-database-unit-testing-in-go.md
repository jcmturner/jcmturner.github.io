---
layout: post
title:  "Database Unit Testing in Go"
date:   2017-08-13 10:30:00 +0100
categories: go
tags: database
published: false
---
I have recently been learning how to interact with databases from Go and I've been looking into unit testing this Go code.

The application I'm writing is using a MariaDB database and ideally I would want to mock out this external dependency to ensure that I concentrate the testing on my code and not on the functioning behaviour of the database.

I know that some people think that such tests are not valid and add no value as there may be differences between the database engine's behaviour and the mock's.  I agree that testing against the actual database engine is required but I don't agree that unit testing with a mocked out database adds no value. 
Firstly, good scientific method dictates that only one variable should be tested at a time. A good mock of the database enables me to get definitive behaviour of the database layer meaning my application code is tested against a well known, static set of behaviours. Therefore if any tests fail I know that the cause of the error must be in my code. 
Secondly, orchestrating an external dependency to be running and in a known state can be very complex. This tends to lead to tests being brittle with a lot of development cycles being wasted on maintaining the test environment. It also leads to slower more complex tests that means feedback into the development cycle takes longer.
Unit testing against a mock enables failures to be spotted more simply and quickly. However a mock does not provide 100% test coverage and I agree that a set of integration tests should be run against an instance of the target database engine. These can follow successful unit testing and would be run less frequently.

With this in mind I considered and ruled out the following options:
* Use Vagrant to run a virtual machine with the MariaDB installed and configured - Difficult to orchestrate for unit testing. This may be something I would do for integration tests.
* Use Docker to run a container of a MariaDB instance - Still quite difficult to orchestrate from within my Go unit tests. This may be something I would do for integration tests.
* Use sqlite as an in-memory database to test against - I looked into this but it was still tricky to set up and didn't give me easy control over behaviour.

I then came across [go-sqlmock](https://github.com/DATA-DOG/go-sqlmock).

It took me a little while to figure out how it worked but in summary its an implementation of a database driver to be used with the standard Go "database/sql" package that you pre-configure with what the driver should expect to see from your code in terms of queries and how the driver should respond to these.

This is very effective to be able to completely control the behaviour of the mock and even simulate database failures to test how your code would handle them.

The rest of this post will outline how I have used it and some of the practices I employed. As I continue to develop with this I hope to add further examples in the future.

