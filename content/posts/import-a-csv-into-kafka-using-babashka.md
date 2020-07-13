+++
date = 2020-07-13T08:15:00Z
description = "A walkthrough of how to import data from a CSV file into a Kafka topic, using a Babashka script"
draft = false
title = "Import a CSV into Kafka, using Babashka"

+++
In life, you don't always get what you want. As developers, we may want all our data in a nice format like [EDN](https://github.com/edn-format/edn "EDN format") or [Transit](https://github.com/cognitect/transit-format "Transit format"), but alas this isn't always possible. Sometimes marketing send you data in a `.docx` file (containing a scan of a photocopy of a screenshot of an HTML table), or perhaps you receive a 5 megabyte Excel spreadsheet from a 3rd party. It's an unfortunate fact of developer life that you have to spend a lot of time massaging data into a usable format.

True to this, I was recently tasked with loading a large CSV file into a Kafka topic. Kafka's CLI tools don't have a built-in way of doing this. You could write a bash script that uses the [Kafka console producer](https://riptutorial.com/apache-kafka/example/27965/kafka-console-producer "Kafka console producer docs"), but using bash for data manipulation is always painful ("what's the syntax for a `for` loop again?"). You could use Python instead, but that's so 90s. If only there were a way to use a powerful, modern, functional language for shell scripting...

This is where [Babashka](https://github.com/borkdude/babashka "Babashka") comes in. Babashka is a derivative of Clojure, designed for shell scripting - it covers the "grey areas of Bash". Clojure is a fantastic language for dealing with data, so Babashka seems like an ideal candidate for our task. In this guide we'll go step-by-step through writing a Babashka script to convert our CSV file to a format we can load into Kafka, which we'll then pipe into the Kafka console producer.

### The Babashka script

Our Babashka script needs to convert each line of the CSV to a key-value format like `message-key::{"foo": 1234}`. This will allow us to pipe the output into the Kafka console producer. Luckily, our CSV has a header row, so we have all the information we need to construct a JSON object for each line. Let's say our CSV looks something like this:

```plaintext
id,email,number-of-pets
1234,alice@gmail.com,3
3456,bob@gmail.com,17
```

Then we want to convert it to a format like this, using `::` as the key/value separator:

    1234::{"id": 1234, "email": "alice@gmail.com", "number-of-pets": 3}
    3456::{"id": 3456, "email": "bob@gmail.com", "number-of-pets": 17}

Let's start by parsing the CSV into a seq of maps:

```clojure
#!/usr/bin/env bb
;; ^^ this tells our shell to use Babashka to run this script

;; read the file path of the CSV from the command line args
(def csv-file-path (first *command-line-args*))

;; read the CSV line-by-line into a data structure
(def csv-data
    (with-open [reader (io/reader csv-file-path)]
    ;; Babashka aliases clojure.data.csv as csv
    (doall (csv/read-csv reader))))

(def headers (first csv-data))
(def body (rest csv-data))

;; For each line in the body, create a map with the headers as the keys
(def values
    (->> body
         (map (partial zipmap headers))
         ;; if you need to do any additional processing on each line, do it here
         ))
```

Now we need to create a seq of formatted key-value pairs. To do this, we need to know which column we should use for the message key, so we'll pass this in as the second command line argument.

```clojure
(def key-field (second *command-line-args*))
     
(def output-lines
    (->> values
         (map #(str (get % key-field) "::" (json/generate-string %)))))
```

We now have a seq of correctly formatted output key-value pairs as `output-lines`. All that's left to do is to print each line to stdout, like so:

```clojure
(doseq [output output-lines]
    (println output))
```

Great, that's all we need for the Babashka script! You can find the script [here](https://gist.github.com/DaveWM/3185481497d32ca623838137e77bd291 "Babashka script gist"), if you'd like to download and run it.

### Setting up Kafka

_If you already have Kafka up and running, you can skip this part._

The easiest way to get started with Kafka is to use the [spotify/kafka](https://hub.docker.com/r/spotify/kafka/ "docker image repository") Docker image, you'll just need [Docker](https://docs.docker.com/get-docker/ "Docker install") installed. To spin it up, run:

     docker run --rm -p 2181:2181 -p 9092:9092 --env ADVERTISED_HOST=localhost -d spotify/kafka

Once it's up and running, we need to create a topic to load our CSV data into. Run this command, which will create a topic called `csv-data`:

    docker run --rm --net=host confluentinc/cp-kafka kafka-topics --create --topic csv-data --replication-factor 1 --partitions 15 --bootstrap-server localhost:9092

Great, you've got a Kafka topic up and running! Now we just need a quick bash script to produce some messages to it...

### Producing to the Kafka topic

We'll now run our Babashka script on a [dummy CSV](/dummy-data.csv "dummy CSV"), then pipe its output into the Kafka console producer. You'll need [Babashka](https://github.com/borkdude/babashka#installation "Babashka install") and [Docker](https://docs.docker.com/get-docker/ "Docker install") installed for this. Here's the one-liner:

```bash
bb csv-to-kafka.clj dummy-data.csv id | docker run --net=host --rm -i confluentinc/cp-kafka kafka-console-producer --broker-list localhost:9092 --topic csv-data --property "parse.key=true" --property "key.separator=::"
```

You can edit the script if you like, perhaps to read from a different CSV or to produce to a different topic. You can now view the messages in your Kafka topic by running:

```bash
docker run --rm --net=host -t edenhill/kafkacat:20190711 -b localhost:9092 -t csv-data -e -f "%k :: %s\n" -q
```

You should see some messages output to the console - job done!

_(Note that when you want to consume this data in an application, you should use the_ [_String Serde_](https://kafka.apache.org/11/javadoc/org/apache/kafka/common/serialization/Serdes.StringSerde.html "String Serde docs") _as the key serde and a_ [_JSON Serde_](https://sachabarbs.wordpress.com/2019/03/14/kafkastreams-custom-serdes/ "JSON Serde blog") _as the value serde)_

### Summary

We went through how to write a short Babashka script to parse a CSV into a key-value pair format, and we then wrote a quick one-liner to run this script and pipe the output to the Kafka console producer. This is a fairly trivial example, but I hope it shows you how Babashka can make shell scripting a bit easier. If you're a glutton for punishment, you can try doing the same thing in pure bash, and see how much harder it is! Thanks for reading.