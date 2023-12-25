---
title:  'Getting Familiar With Nginx'
date:  2023-12-25T22:28:40+01:00
draft:  true
---
## Introduction
Some time ago I've stared working on web app in Go. I've been using standard `net/http` library to handle all requests. I have also implemented some basic logging and then I realized that I've written quite a bit code that not necessary is related with my app, but it does basic http server stuff. So I've asked myself: "Do I need use some http server like for Python web app? Or can I implement all with `net/http` Go library. I've started to looking for some answers and have found many statements that I don't need http server and can implement everything in Go, but most of the times using http sever like nginx can save me a lot of time and issues. Of course for learning purposes it is good idea to implement all by yourself. You will face many issues and rabbit holes by going this way and it will give you experience and understating, but if you want to deliver some app you can leverage existing solutions that solve all those problems.
Keeping that in mind I've decided to learn more about nginx. I want to have better understating what it can provide and how I can leverage it. Currently I have some experience with nginx configuration, but I don't feel confident and I definitely not aware of all its features. The purpose of this post is to document my familiarizing with nginx.
