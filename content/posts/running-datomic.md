---
title: "The Cheapskate's Guide to Running Datomic"
date: 2018-04-02T02:43:48+01:00
draft: true
---

First off, welcome to my brand new blog. I'm a Clojure programmer by day, so this blog will focus pretty much entirely on Clojure. My aim with this blog is to document solutions for any rare problems I come across.

I was inspired to start the blog by a [this post on Medium](https://medium.com/@jiyinyiyong/clojurians-please-share-your-knowledges-with-blogs-c674503f54a). The post talks about how Clojure can be hard for beginners, due to the lack of step-by-step guides for common problems. It's from the point of view of a Clojure beginner, but I believe a lot of it applies to any Clojure programmer, no matter how experienced. I've been learning Clojure for a few years, and when I have a problem I still often find myself sifting through Google Groups/Clojurians Slack/GitHub issues, trying to find someone who's previously encountered the same issue. 

In this post, I'll cover something that I've been wrestling with recently - running [Datomic](https://www.datomic.com/) in the cloud as cheaply as possible. 

## What's the Problem?

Datomic is an immutable database, created by Cognitect. I won't go into detail about Datomic here, or why you should use it (but you definitely should). If you want to learn more about Datomic works, I'd recommend watching Rich Hickey's talk ["Database as a Value"](https://www.youtube.com/watch?v=EKdV1IgAaFc) - if that doesn't convince you to use Datomic, nothing will. 

The problem I had, is that running Datomic on AWS is pretty expensive. I initially tried using the scripts bundled with Datomic, which create and run a CloudFormation stack. However, I found after less than a month that I'd been billed over $45, which was too much for me. I then tried using Datomic Cloud, but I again found it was too expensive - I was charged $25 for less than a week. When you're just starting out with Datomic, you don't want to spend this amount of money. You just want something cheap (or free), regardless of how slow it is.

## Solution Overview

I wanted to run Datomic in a docker container, for a couple of reasons:

* It's easy to run locally on your dev machine
* Portability - it should make it easier to change to a different cloud provider, if you ever need to

I looked at several platforms for Docker container hosting, including [hyper.sh](https://hyper.sh/) and [sloppy.io](https://sloppy.io/pricing/). Datomic realistically requires 2GB of RAM, which would cost around $15-20 per month. It actually worked out cheaper to rent a whole virtual machine, and run a Docker container on it. The platform I eventually landed on was [DigitalOcean](https://www.digitalocean.com). A VM (droplet in their terminology) with 2GB of RAM costs $10 per month, plus they give you $100 credit to get started. 

Datomic is a bit different to other databases, in that it allows you to choose which underlying storage you want to use. You can choose between AWS's DynamoDB, an SQL database or Cassandra. I decided to go with PostgresQL, because there are plenty of hosting options. I went with [Heroku Postgres](https://elements.heroku.com/addons/heroku-postgresql), because I was already using Heroku, and they have a free tier. I've never used any other hosted SQL service, so I won't recommend any others, but there are plenty out there.

# Step-by-step Guide

* We'll start by setting up PostgresQL on Heroku. 




