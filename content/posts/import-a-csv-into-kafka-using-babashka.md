+++
date = 2020-07-06T23:00:00Z
description = "A walkthrough of how to import data from a CSV file into a Kafka topic, using a Babashka script"
draft = true
title = "Import a CSV into Kafka, using Babashka"

+++
In life, you don't always get what you want. We may want all our data in EDN or Transit format, but alas this isn't always possible. Sometimes marketing send you data in a .docx file (a scan of a photocopy of a screenshot taked with a phone camera), or you get a 5 megabyte XML file from a 3rd party. It's an unfortunate fact of developer life that you have to spend a lot of time massaging data into a usable format.

I recently ran into a situation where I had to load a large CSV file into a Kafka topic. Confluent, the maintainers of Kafka, don't provide any way of doing this in Kafka's CLI tools. You can use a combination of a bash script and the Kafka console producer to do this, but using bash for data manipulation is always painful. You can use Python instead, but that's so 90s. If only there were a way to use a powerful, modern, functional language for shell scripting...

This is where [Babashka](https://github.com/borkdude/babashka "Babashka") comes in. Babashka is a derivative of Clojure, designed for shell scripting - it covers the "grey areas of Bash". Clojure is a fantastic language for dealing with data, so Babashka seems like an ideal candidate. In this guide we'll write a Babashka script to convert our CSV file to a format we can load into Kafka, which we'll then pipe into the Kafka Console Producer.