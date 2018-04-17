---
title: "The Cheapskate's Guide to Running Datomic"
date: 2018-04-09T23:00:00+01:00
draft: false
---

First off, welcome to my brand new blog. I'm a Clojure programmer by day, so this blog will focus pretty much entirely on Clojure (as you probably guessed from the title). My main aim with this blog is to document step-by-step solutions for any difficult, or rare, problems I come across.

I was inspired to start the blog by [this post on Medium](https://medium.com/@jiyinyiyong/clojurians-please-share-your-knowledges-with-blogs-c674503f54a). The post talks about how Clojure can be hard for beginners, due to the lack of step-by-step guides for common problems. It's from the point of view of a Clojure beginner, but I believe a lot of it applies to any Clojure programmer, no matter how experienced. I've been learning Clojure for a few years, and when I have a problem I still often find myself sifting through Google Groups/Clojurians Slack/GitHub issues for different bits of the solution, then attempting to put all the pieces together into something that works for me. This can be quite difficult, especially for beginners, and sometimes it's preferable to just follow an opinionated tutorial. 

One thing I struggled with recently is making a cheap [Datomic](https://www.datomic.com/) setup, so I thought this would be a good subject for my first post. I'll give you a step-by-step guide of how to get Datomic up and running for $10 per month.

## What's the Problem?

Datomic is an immutable database, created by Rich Hickey and Cognitect. I want to focus on how to get Datomic up and running, so I'll assume you know a bit about Datomic and how it works. I won't go into detail about how it works, or why you should use it (but you definitely should). If you've never heard of Datomic before, and you want to learn more about the philosophy behind it, and the problems it solves, I'd recommend watching Rich Hickey's talk ["Database as a Value"](https://www.youtube.com/watch?v=EKdV1IgAaFc) - if that doesn't convince you to use it, nothing will. 

First, some background. I had written an app, the backend of which used Datomic. I wanted to set up a very basic "production" environment for alpha testing. I was looking to run Datomic cheaply, but in such a way that I could scale it up without too much effort. This ruled out running Datomic in [dev mode](https://docs.datomic.com/on-prem/dev-setup.html).

I initially tried using the scripts bundled with Datomic, which create and run a CloudFormation stack on AWS. However, I found after less than a month that I'd been billed over $45. I then tried using Datomic Cloud instead, but I again found it was quite expensive - I was charged $25 for less than a week. When you're just starting out with an app, you don't want to be paying this much (or at least I didn't). I therefore set out to create a more "affordable" Datomic setup.

## Solution Overview

Datomic is a bit different to most databases, in that the underlying storage is completely decoupled from the process which writes to it (called the "transactor"). I wanted to run the Datomic transactor in a docker container, for a few reasons:

* It makes it easier to change to a different hosting provider, if you ever need to
* It makes creating new environments easier
* You don't have to worry about having all the correct dependencies installed (e.g. the correct java version)
* It's more predictable - if the transactor works correctly when you run it in a container, you can be sure

Luckily, we don't have to build a Docker image from scratch, there's a [base image on GitHub](https://github.com/pointslope/docker-datomic), courtesy of [PointSlope](https://www.pointslope.com/).

I looked at several platforms for Docker container hosting, including [hyper.sh](https://hyper.sh/) and [sloppy.io](https://sloppy.io/). Since the base Docker image above requires a minimum of 1GB or memory, I needed more than 1GB of RAM, which would've cost around $15-20 per month. It actually worked out cheaper to rent a whole virtual machine, and run a Docker container on it. The platform I eventually landed on was [DigitalOcean](https://www.digitalocean.com). A VM (droplet in their terminology) with 2GB of RAM costs $10 per month, plus they give you $100 credit to get started. 

You have several choices of underlying storage. You can use AWS's DynamoDB, any SQL database or Apache Cassandra. I decided to go with PostgreSQL, because there are plenty of cheap hosting options. I went with [Heroku Postgres](https://elements.heroku.com/addons/heroku-postgresql), because I was already using Heroku, and they have a free tier. I've never used any other hosted SQL service, so I can't recommend any others, but there are plenty out there.

# Step-by-step Guide

_Note: when you see something in square brackets (like "[db host]"), you need to fill it in_

## Prerequisites

* You'll need an account for [https://my.datomic.com](https://my.datomic.com), and a license key for Datomic
* You'll also need a [Docker Hub](https://hub.docker.com) account to host your Datomic transactor Docker image

### Setting up PostgreSQL

* First, make sure you have an app set up on Heroku. If you don't know how to do this, follow [this guide](https://devcenter.heroku.com/articles/getting-started-with-clojure#introduction)
* Go to [the Heroku Postgres addon page](https://elements.heroku.com/addons/heroku-postgresql)
* Click "Install Heroku Postgres", and select your app
* Select the "Hobby Dev" tier and click "Provision"

A Postgres instance should now be running. We'll need the connection details when we set up the Datomic transactor, so let's note them down now. On Heroku, go to your app, then the "Resources" tab, then click on "Heroku Postgres". 

![Resources tab](https://i.imgur.com/EbIuFOQ.png)

Now click on the "Settings" tab -> database credentials. You'll need the host, database, user, port and password later on. 

![Postgres settings tab](https://i.imgur.com/4gxRlcl.png)

### Initialising the DB for Datomic

Before we can use our DB with Datomic, we need to set up a table that Datomic requires, `datomic_kvs`. To do this, follow these steps:

* Run `sudo docker run -p 80:80 -e "PGADMIN_DEFAULT_EMAIL=[your email]" -e "PGADMIN_DEFAULT_PASSWORD=[any password]" -d dpage/pgadmin4`
* Go to [http://localhost](http://localhost) in your web browser, and log in using the email and password you specified
* Right click on "servers" in the left hand menu, then select Create -> Server
* Enter the connection details you noted down earlier (or look them up again in Heroku, because you didn't note them down like I told you to :P)
* Once you've connected to the server, find your DB in the "Databases" list (protip: use ctrl+f), right click on it, and select "Query Tool"
* Paste in this script (which is bundled with Datomic), and run it:

```
CREATE TABLE datomic_kvs
(
 id text NOT NULL,
 rev integer,
 map text,
 val bytea,
 CONSTRAINT pk_id PRIMARY KEY (id )
)
WITH (
 OIDS=FALSE
);
ALTER TABLE datomic_kvs
 OWNER TO postgres;
GRANT ALL ON TABLE datomic_kvs TO postgres;
GRANT ALL ON TABLE datomic_kvs TO public;
```

### Creating a Docker Image

In this step, we'll create a Docker image for our Datomic transactor, using [pointslope/datomic-pro-starter](https://hub.docker.com/r/pointslope/datomic-pro-starter/) as the base image. 

* Create a new directory called `datomic-docker`, and `cd` to it
* Run `echo "[email]:[datomic download key]" >> .credentials` (you can find your download key [here](https://my.datomic.com))
* Copy the following to `config/transactor.properties` (you need to fill in the bits in square brackets):

```
protocol=sql
host=[DigitalOcean droplet IP address]
port=4334
alt-host=localhost
sql-url=jdbc:postgresql://[postgres host]:[postgres port]/[postgres DB name]
sql-user=[postgres user]
sql-password=[postgres password]
sql-driver-class=org.postgresql.Driver
sql-driver-params=ssl=true;sslfactory=org.postgresql.ssl.NonValidatingFactory;

license-key=[datomic license key]

# Recommended settings for -Xmx1g usage, e.g. dev laptops.
memory-index-threshold=32m
memory-index-max=256m
object-cache-max=128m
```

* Copy this to `Dockerfile`:

```
FROM pointslope/datomic-pro-starter:0.9.5561
MAINTAINER [your name] "[your email]"
CMD ["config/transactor.properties"]
```

* Run `sudo docker build -t [docker hub username]/[image name] .`
* Run `sudo docker push [docker hub username]/[image name]`

### Creating a DigitalOcean Droplet

* Go to [DigitalOcean](https://www.digitalocean.com/), and create an account
* Click on "Create", then "Droplets"
* Go to the "One-click apps" tab 
* Choose "Docker 17.09.9-ce on 16.04", and the 2GB/1vCPU size
* Click "Create"
* Once you've received the username and password via email, you should be able to SSH to the droplet by running `ssh [user]@[droplet IP]` then entering the password

For security, it's recommended to set up SSH keys to access your droplet, rather than using a username and password. If you want to do this, just follow [this guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2).

### Starting the Datomic Transactor

* SSH to your droplet by running `ssh root@[droplet ip address]`, then entering the password when prompted
* To start your transactor, run `docker run -p 4334:4334 -p 4335:4335 -v data:/opt/datomic-pro-0.9.5561/data --net=host --detach [docker hub username]/[image name]`
* To verify your container is running, run `docker ps` - you should see a single running container. Then, run `curl [droplet ip]:4334`, and you should see "Empty reply from server".

Congratulations, you've now got your own instance of Datomic running in the cloud! Now you just need to add it to your app.

### Using Datomic in your app

There are 2 possible APIs you can use with Datomic: client or peer. I'll use the peer API here, mainly because it's easier to set up.

* If you're making a new app, run `lein new [app name]`
* Add Datomic as a dependency in `project.clj`, by adding `[com.datomic/datomic-pro "0.9.5561"]` to `:dependencies`. Note that the transactor we're running is version 0.9.5561, so it's best to use this version in your app as well, although later versions do seem to work.
* Add the following code to your app:

```
(ns my-app.core
  (:require [datomic.api :as d]))

(def db-uri "datomic:sql://[datomic db name]?jdbc:postgresql://[postgres db host]:[postgres port]/[postgres db name]?user=[postgres user]&password=[postgres password]&ssl=true&sslfactory=org.postgresql.ssl.NonValidatingFactory")

(d/create-database db-uri)

(def connection (d/connect db-uri))
```
(Note that the Datomic DB name can be whatever you want.)

That's it! You now have a connection to your Datomic DB.

![Good job!](https://media.giphy.com/media/Yb3d5B1zwuhCo/giphy.gif)

## Where to go from here

If you're new to Datomic, have a look through the [official getting started guide](https://docs.datomic.com/on-prem/peer-getting-started.html). You should also check out [Day of Datomic](https://github.com/Datomic/day-of-datomic), and [Learn Datalog Today](http://www.learndatalogtoday.org/).

If you're writing a large app, I'd definitely recommend using either [Mount](https://github.com/tolitius/mount), [Component](https://github.com/stuartsierra/component), or [Integrant](https://github.com/weavejester/integrant) to manage your connection state.

## Wrapping up

I hope you've found this post useful. If you've spotted any inaccuracies (euphemism for "glaring security flaws"), or you'd just like to congratulate me on what an excellent post this is, feel free to email me at [dwmartin41@gmail.com](mailto:dwmartin41@gmail.com). Thanks for reading.
