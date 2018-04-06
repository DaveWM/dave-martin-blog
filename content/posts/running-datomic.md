---
title: "The Cheapskate's Guide to Running Datomic"
date: 2018-04-02T02:43:48+01:00
draft: false
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

Datomic is a bit different to other databases, in that it allows you to choose which underlying storage you want to use. You can choose between AWS's DynamoDB, an SQL database or Cassandra. I decided to go with PostgreSQL, because there are plenty of hosting options. I went with [Heroku Postgres](https://elements.heroku.com/addons/heroku-postgresql), because I was already using Heroku, and they have a free tier. I've never used any other hosted SQL service, so I won't recommend any others, but there are plenty out there.

# Step-by-step Guide

## Prerequisites

* You'll need an account for [https://my.datomic.com](https://my.datomic.com), and a license key for Datomic
* You'll also need a [Docker Hub](https://hub.docker.com) account to host your Datomic transactor Docker image

### Setting up PostgreSQL

* First, make sure you have an app set up on Heroku. If you don't know how to do this, follow [this guide](https://devcenter.heroku.com/articles/getting-started-with-clojure#introduction)
* Go to [the Heroku Postgres addon page](https://elements.heroku.com/addons/heroku-postgresql)
* Click "Install Heroku Postgres", and select your app
* Select the "Hobby Dev" tier and click "Provision"

A Postgres instance should now be running. We'll need the connection details when we set up the Datomic transactor, so let's note them down now. On Heroku, go to your app -> resources tab -> Heroku Postgres -> settings tab -> database credentials. You'll need the host, database, user, port and password later on. 

### Initialising the DB for Datomic

Before we can use our DB with Datomic, we need to run create the `datomic_kvs` table. Just follow these steps:

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

In this step, we'll create a Docker image for our Datomic transactor. 

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
* You should now receive an email with the username and password for the droplet
* You can now SSH to the droplet by running `ssh root@[droplet IP]`, then entering the password

For security, it's recommended to set up SSH keys to access your droplet, rather than using a username and password. If you want to do this, just follow [this guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2).

### Starting the Datomic Transactor

* SSH to your droplet by running `ssh root@[droplet ip address]`, then entering the password when prompted
* To start your transactor, run `docker run -p 4334:4334 -p 4335:4335 -v data:/opt/datomic-pro-0.9.5561/data --detach [docker hub username]/[image name]`
* To verify your container is running, run `docker ps` - you should see a single running container

Congratulations, you've now got your own instance of Datomic running in the cloud!

