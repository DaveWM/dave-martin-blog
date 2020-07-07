+++
date = 2020-07-06T23:00:00Z
description = "A walkthrough of how to import data from a CSV file into a Kafka topic, using a Babashka script"
draft = true
title = "Import a CSV into Kafka, using Babashka"

+++
In life, you don't always get what you want. We may want all our data in EDN or Transit format, but alas this isn't always possible. Sometimes marketing send you data in a .docx file (a scan of a photocopy of a screenshot taked with a phone camera), or you get a 5 megabyte XML file from a 3rd party. It's an unfortunate fact of developer life that you have to spend a lot of time massaging data into a usable format.

I recently ran into a situation where I had to load a large CSV file into a Kafka topic. Confluent, the maintainers of Kafka, don't provide any way of doing this in Kafka's CLI tools. You can use a combination of a bash script and the Kafka console producer to do this, but using bash for data manipulation is always painful. You can use Python instead, but that's so 90s. If only there were a way to use a powerful, modern, functional language for shell scripting...

This is where [Babashka](https://github.com/borkdude/babashka "Babashka") comes in. Babashka is a derivative of Clojure, designed for shell scripting - it covers the "grey areas of Bash". Clojure is a fantastic language for dealing with data, so Babashka seems like an ideal candidate. In this guide we'll write a Babashka script to convert our CSV file to a format we can load into Kafka, which we'll then pipe into the Kafka Console Producer.

### The Babashka script

Our Babashka script needs to convert each line of the CSV to a key-value format like `key::{"value": true}`. This will allow us to pipe the output into the Kafka Console Producer, using `::` as the key-value separator. Luckily, our CSV has a header row, so we have all the information we need to construct a JSON object for each line. Our CSV looks like this:

    Id,Foo,Bar
    1234,"foo","bar"
    3456,"baz","baz"

And we want to convert it to a format like this:

    1234::{"id": 1234, "foo": "foo", "bar": "bar"}
    3456::{"id": 3456, "foo": "baz", "bar": "baz"}

Let's start by parsing the CSV into a seq of maps:

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
           (map (partial zipmap headers))
           ;; if you need to do any additional processing on each line, do it here
           ))

Now we can create a seq of formatted key-value pairs (separated by `::`). To do this, we need to know which field we should use for the key, so we'll pass this in as the second command line argument.

     (def key-field (second *command-line-args*))
     
     (def output-lines
        (->> data
             (map #(str "\"" (get % key-field) "\"" "::" (json/generate-string %)))))

We now have a seq of correctly formatted output key-value pairs as `output-lines`. All that's left to do is to print each line to stdout, like so:

    (doseq [output output-lines]
        (println output)))

Great, that's all we need for the Babashka script! You can find the script [here](https://gist.github.com/DaveWM/3185481497d32ca623838137e77bd291 "Babashka script gist"), if you'd like to download and run it. Now we just need a quick bash one-liner...

### The bash one-liner

We'll run our Babashka script with the correct arguments (path to the CSV file + the name of the key field), then just pipe it into the Kafka Console Producer. You can do this like so (you'll need to update the bits in square brackets according to your Kafka setup):

    bb csv-to-kafka.clj [path to csv] [key field name] | kafka-console-producer --broker-list [Kafka broker url, usually ends with :9092] --topic [topic name] --property "parse.key=true" --property "key.separator=::"

Or alternatively if you don't have the Kafka CLI tools installed, you can run them in a Docker container:

    bb csv-to-kafka.clj [path to csv] [key field name] | docker run --net=host -i confluentinc/cp-kafka kafka-console-producer --broker-list [Kafka broker url, usually ends with :9092] --topic [topic name] --property "parse.key=true" --property "key.separator=::"

That's it! You should now see the CSV data in your Kafka topic.

### Summary

We went through how to write a short Babashka script to parse a CSV into a key-value pair format, and we then wrote a quick one-liner to run this script and pipe the output to the Kafka Console Producer. This is a fairly trivial example, but I hope it shows you how Babashka can make shell scripting just a bit easier. If you're a masochist, you can try doing the same thing in bash, and see how much harder it is!