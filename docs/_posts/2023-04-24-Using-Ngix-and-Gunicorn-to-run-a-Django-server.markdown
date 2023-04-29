---
layout: post
title: "Using Nginx and Gunicorn to run a Django server"
date: 2023-04-24 11:45:00 +0000
tags: servers
description: "An Nginx front facing server with a Gunicorn backend serving a Django server"
---
In this post, I will illustrate how to create a production-ready Django-based server using Nginx web server and Gunicorn WSGI server. Highlevel overview of this implementation is as follows:

![](/assets/post_images/wsgi_web_highlevel.png)