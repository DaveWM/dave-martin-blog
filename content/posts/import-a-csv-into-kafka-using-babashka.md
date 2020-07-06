+++
date = 2020-07-06T23:00:00Z
description = "A walkthrough of how to import data from a CSV file into a Kafka topic, using a Babashka script"
draft = true
title = "Import a CSV into Kafka, using Babashka"

+++
In life, you don't always get what you want. We may want all our data in EDN or Transit format, but alas this isn't always possible. Sometimes marketing send you data in a .docx file (a scan of a photocopy of a screenshot taked with a phone camera), or you get a 5 megabyte XML file from a 3rd party. It's an unfortunate fact of developer life that you have to spend a lot of time massaging data into a usable format.

I recently ran into a situation where I had to load a large CSV file into a Kafka topic. Confluent, the maintainers of Kafka, don't provide any way of doing this in Kafka's CLI tools. You can use a combination of a bash script and the Kafka console producer to do this, but using bash for data manipulation is always painful. You can use Python instead, but that's so 90s. If only there were a way to use a powerful, modern, functional language for shell scripting...

This is where [Babashka](https://github.com/borkdude/babashka "Babashka") comes in. Babashka is a derivative of Clojure, designed for shell scripting - it covers the "grey areas of Bash". Clojure is a fantastic language for dealing with data, so Babashka seems like an ideal candidate. In this guide we'll write a Babashka script to convert our CSV file to a format we can load into Kafka, which we'll then pipe into the Kafka Console Producer.

### Writing the Babashka script

Our Babashka script needs to convert each line of the CSV to a key-value format like `key::{"value": true}`. This will allow us to pipe the output into the Kafka Console Producer, using `::` as the key-value separator. Luckily, our CSV has a header row, so we have all the information we need to construct a JSON object for each line. Our CSV looks like this:

    Id,Foo,Bar
    1234,"foo","bar"
    3456,"baz","baz"

And we want to convert it to a format like this:

    1234::{"id": 1234, "foo": "foo", "bar": "bar"}
    3456::{"id": 3456, "foo": "baz", "bar": "baz"}

Let's start by calculating the values:

    #!/usr/bin/env bb
    ;; ^^ this tells our shell to use Babashka to run this script
    
    ;; read the file path of our CSV from the command line args
    (def csv-file-path (first *command-line-args*))
    
    ;; read the CSV line-by-line into a data structure
    (def csv-data
      (with-open [reader (io/reader csv-file-path)]
        (doall (csv/read-csv reader))))
    
    (def headers (first csv-data))
    (def body (rest csv-data))
    
    ;; For each line in the body, create a map with the headers as the keys
    (def values
      (->> body
           (map (partial zipmap headers))))