---
title: "Kafka Streams, the Clojure way"
date: 2019-08-14T00:00:00+01:00
draft: true
---

In this blog post, I'll walk you through how to create a Kafka Streams app in the style of idiomatic Clojure. I won't assume any knowledge of Kafka or Kafka Streams, but if you've never heard of them before I'd check out Confluent's introduction to Kafka Streams here (TODO - link). Kafka can be thought of as a persistent, distributed message queue, mainly targeted at JVM languages. All your messages are stored in "topics", and the Kafka API gives you means to produce to and consume from these topics. Kafka Streams is an abstraction on top of Kafka, which treats topics as a stream of data onto which you can apply transformations (map, filter, etc.). You can think of these streams as being like RxJS Observables. It also gives you a way of aggregating topics. The Java API for Kafka Streams is very powerful, but has a few drawbacks. It gives you a mutable `KStreamsBuilder` object, on which you call methods like `.map(...)` and `.filter(...)`. You use this builder object to build a "topology", which can be thought of as the actual running Kafka Streams application. We'll develop an application in this style using the Jackdaw library (a Kafka Streams wrapper), and then evolve it into a more Clojure-ish, data-driven style. All the code I'm going to show you is in this walkthrough repo (TODO - create repo + link here), so clone it now if you'd like to follow along. 

## A Simple Example

Let's imagine that you work at Fidget-no-more Incorporated, the world's largest purveyor of fidget spinners (or whatever the kids are buying nowadays). Your website currently records every purchase in a DB, but other departments are finding it difficult to write applications that react when a purchase is made. The sales team would like their apps to somehow be notified when a large purchase is made, so they can send the user a personalised thank you email. This sounds like the perfect job for Kafka! Here's what we'll do:
1. On every purchase, produce a message to the "purchase-made" Kafka topic.
2. Create a Kafka Streams app that:
    * Reads from this topic
    * Filters for large purchases (above £100)
    * Removes any extraneous fields from each message (the sales team only need a user id and an amount)
    * Writes to the “large-transaction-made" topic
Before we start, we need to start up a Kafka broker (TODO - link + definition). The easiest way to do this is to spin up a fast-data-dev (TODO - link) Docker container. Do this by running the following command (you’ll need Docker (TODO - link) installed): 

```
docker run --rm --net=host landoop/fast-data-dev
```

Once you've done this, let’s produce a few messages to the `purchase-made` topic. To do this, we’ll first need to create the "purchase-made" topic. Here’s the code:

```clojure
;; First, we need to define the Kafka config, so we can connect to the Kafka broker
(def kafka-config
  {"application.id" "kafka-streams-the-clojure-way"
   "bootstrap.servers" "localhost:9092"
   "default.key.serde" "jackdaw.serdes.EdnSerde"
   "default.value.serde" "jackdaw.serdes.EdnSerde"
   "cache.max.bytes.buffering" "0"})


;; define the topic configs
(def serdes
  {:key-serde (serde)
   :value-serde (serde)})


(def purchase-made-topic
  (merge {:topic-name "purchase-made"
          :partition-count 1
          :replication-factor 1
          :topic-config {}}
         serdes))


(def large-transaction-made-topic
  (merge {:topic-name "large-transaction-made"
          :partition-count 1
          :replication-factor 1
          :topic-config {}}
         serdes))


;; create the "purchase-made" and "large-transaction-made" topics
(ja/create-topics! admin-client [purchase-made-topic large-transaction-made-topic])
```


We now need to produce some dummy messages to the "purchase-made" topic. To do this, we’ll create a `Producer` object and then pass it to the `produce!` function.  In production code for an API, you’d most likely create a `Producer` when the application starts up, then call `produce!` in the appropriate response handler function. For now, let’s just pretend that we've made a few purchases by producing some messages to the "purchase-made" topic from the repl. These messages are going to look like this:

```
{:id 1
 :user-id 1234
 :amount 20
 :quantity 5}
```

Clone the walkthrough repo if you haven't already, and start up your repl. Run the following commands in the `kafka-streams-the-clojure-way.core` namespace: 

```
(defn make-purchase! [amount]
  (let [purchase-id (rand-int 10000)
        user-id     (rand-int 10000)
        quantity    (inc (rand-int 10))]
    (with-open [producer (jc/producer kafka-config serdes)]
      @(jc/produce! producer purchase-made-topic purchase-id {:id purchase-id
                                                              :amount amount
                                                              :user-id user-id
                                                              :quantity quantity}))))


;; Make a few dummy purchases
(make-purchase! 10)
(make-purchase! 500)
(make-purchase! 50)
(make-purchase! 1000)
```

 You should see in the Landoop UI that a message appears in the topic (TODO - link to fast-data-dev topic UI). Now we have a topic with some dummy data in it, we can start writing our Kafka Streams topology:

```
(defn build-topology [builder]
  (-> (js/kstream builder purchase-made-topic)
      (js/filter (fn [[_ purchase]]
                   (<= 100 (:amount #spy/p purchase))))
      (js/map-values (fn [purchase]
                       (select-keys purchase [:amount :user-id])))
      (js/to large-transaction-made-topic)))


(defn start! []
  (let [builder (js/streams-builder)]
    (build-topology builder)
    (doto (js/kafka-streams builder kafka-config)
      (js/start))))
```

Run `(start!)` in the repl to start the topology. You should shortly see messages appear on the `large-transaction-made` topic. Magic! Try producing some more messages to the `purchase-made` topic, and your topology should pick them up and process them immediately. When you're ready to move on, run `(stop!)` to stop the topology.


## Introducing Transducers


We’ve made a good start, but there are a few problems with our code at the moment. One is that our code is difficult to test. We can’t easily test the logic of the topology as a whole, because it’s tied into the Kafka Streams API. You can use the TopologyTestDriver class to run your topology in memory, but this can be quite cumbersome. It would be much nicer if we had some pure functions to test. Secondly, our code isn’t composable. If you have 2 separate topologies, there is no easy way of merging them together. Thirdly, the code is tied to a specific context - Kafka Streams. If you want to re-use the same logic for transforming, for example, a core.async channel you’d be forced to re-write it. 


To alleviate these problems, we’re going to use transducers (TODO - link). Transducers are "a powerful and composable way to build algorithmic transformations that you can reuse in many contexts”. If you’re not familiar with them already, you may want to check out this introductory blog post (TODO - link). Let’s write a transducer that captures the logic of our topology:

```
(def purchase-made-transducer
  (comp
    (filter (fn [[_ purchase]]
              (<= 100 (:amount purchase))))
    (map (fn [[key purchase]]
           [key (select-keys purchase [:amount :user-id])]))))
```

We can easily test the transducer in the repl like this:

```
(into []
        purchase-made-transducer
        [[1 {:purchase-id 1 :user-id 2 :amount 10}]
         [3 {:purchase-id 3 :user-id 4 :amount 500}]])
```

We've isolated the logic of our topology as a pure function, which is much easier to test than a Now we'll update our topology to use this transducer. We're going to introduce the `transduce-stream` function here - you don't need to know exactly how it works, only that it applies a transducer to a `KStream`.

```
(defn build-topology-with-transducer [builder]
  (-> (js/kstream builder purchase-made-topic)
      (transduce-stream purchase-made-transducer)
      (js/to large-transaction-made-topic)))
```

Great, this seems to have made our code more testable, flexible, and composable. 


## A more complicated Example


Unfortunately, your boss now has another requirement (as they always do). Your company has just launched a new way of buying fidget spinners - “The Humble Spinner Bundle”. Your customers pay whatever they think is fair for a bundle of 10 (ten!) fidget spinners. This is done through a completely different site, but luckily the team in charge of building it have created a Kafka topic for you, onto which they’ll publish all the purchases of the humble bundle - `humble-donation-made`. The sales team would like to send a congratulatory email to customers who pay a large amount for the bundle, in the same way as they do for regular purchases. Therefore, you’ve been tasked with merging this topic into your existing topology. Unfortunately, the team building the `humble-donation-made` topic have come up with a message schema that's slightly different to `purchase-made`.  `humble-donation-made` messages look like this:
 ```
{:user-id 1234
 :donation-amount-cents 1000
 :donation-date "2019-01-02"}
 ```


We’ll need to apply a transducer to the `humble-donation-made`, then use the Kafka Streams `merge` function to merge it with the `purchase-made` stream. Let's start with the transducer for the `humble-donation-made` topic:
```
(def humble-donation-made-transducer
  (comp
    (filter (fn [[_ donation]]
              (<= 10000 (:donation-amount-cents donation))))
    (map (fn [[key donation]]
           [key {:user-id (:user-id donation)
                 :amount (int (/ (:donation-amount-cents donation) 100))}]))))
```




Now for the topology itself, this is what the code looks like:

```
(defn more-complicated-topology [builder]
  (js/merge
    (-> (js/kstream builder purchase-made-topic)
        (transduce-stream purchase-made-transducer))
    (-> (js/kstream builder humble-donation-made-transducer)
        (transduce-stream humble-donation-made-transducer))))
```


This will work, but we’ve re-introduced all the problems we had before we started using transducers. Our code is now less testable, less composable, and less portable. As Clojure programmers, we know exactly what to do when we’re confronted with this problem - express it as data! If only there were a library that could help us out…


## Introducing Willa

Willa (TODO - link) is just such a library. It allows you to express your topology as data and functions, rather than using the mutable KStreamsBuilder API. Full disclosure - I’m the author of Willa, so if you’re looking for an unbiased critique of it, I’d just stop reading now. Let’s see how it would affect our code. We’ll start by defining all our topics and streams. Willa calls these things “entities”, and you define them like so:

```
(def entities
  {:topic/purchase-made (assoc purchase-made-topic ::w/entity-type :topic)
   :topic/humble-donation-made (assoc humble-donation-made-topic ::w/entity-type :topic)
   :topic/large-transaction-made (assoc large-transaction-made-topic ::w/entity-type :topic)




   :stream/large-purchase-made {::w/entity-type :kstream
                                ::w/xform purchase-made-transducer}
   :stream/large-donation-made {::w/entity-type :kstream
                                ::w/xform humble-donation-made-transducer}})
```

So far so good. We now need to define how our topics and streams relate to each other - how the data flows through our topology. Willa models a topology as a directed acyclic graph (DAG) of entities. You express this as a vector of tuples. Each tuple has 2 elements, each an entity id, and represents an edge on the DAG. Willa calls this a “workflow”. We can express our workflow like so:

```
(def workflow
  [[:topic/purchase-made :stream/large-purchase-made]
   [:topic/humble-donation-made :stream/large-donation-made]
   [:stream/large-purchase-made :topic/large-transaction-made]
   [:stream/large-donation-made :topic/large-transaction-made]])
```

Putting it together:
```
(def topology
  {:workflow workflow
   :entities entities})
```


That's all we need to do to completely specify our topology. One nice advantage of doing it this way is that we can now visualise our topology. Run this command in the repl:
```
(wv/view-topology topology)
```


You should see a diagram like this:

![Topology Diagram](/images/topology.png)


To get it running, we just need a few lines of code to compile Willa’s representation of our topology into an actual Kafka Streams topology:

```
(js/start
    (w/build-topology! (js/streams-builder) topology))
```


So now we’ve completely specified our topology as data structures and transducers. What does that give us, other than being able to brag to other developers about how decomplected our code is? (Note - please don’t do that). As we saw earlier, we’re able to test each transducer in isolation, without knowing anything about Kafka Streams.


We can also quickly experiment with the topology, without interacting with Kafka at all. In Willa, this is called an "experiment". You can do this by running this in the repl:

```
(def experiment-results
  (we/run-experiment topology
                     {:topic/purchase-made [{:key 1
                                             :value {:id 1
                                                     :amount 200
                                                     :user-id 1234
                                                     :quantity 100}}]
                      :topic/humble-donation-made [{:key 2
                                                    :value {:user-id 2345
                                                            :donation-amount-cents 15000
                                                            :donation-date "2019-01-02"}}]}))


;; Visualise experiment result
(wv/view-topology experiment-results)


;; View results as data
(->> experiment-results
     :entities
     (map (fn [[k v]]
            [k (::we/output v)]))
     (into {}))
```

The experiment results should look like this:

![Experiment Results Diagram](/images/experiment.png)


You can also verify that the topology is valid in the repl, using Clojure Spec. You can do this by running:

```
(s/explain ::ws/topology topology)


;; What happens with an invalid topology?
(s/explain ::ws/topology
           ;; introduce a loop in our topology, which is not allowed
           (update topology :workflow conj [:topic/large-transaction-made :topic/purchase-made]))
```

The real advantage of Willa is that it relies on standard Clojure data structures as much as possible. This allows you to do things like serialise your topology as EDN, programatically manipulate it, and also query it - for example, try writing a function to count the number of topics involved in a topology.


## Summary


I hope that’s given you a brief overview of how to write a basic Kafka Streams topology. If you’re interested in learning more, I’d recommend starting here (TODO - link to good book). We’ve barely scratched the surface of Kafka Streams. We haven’t even touched on KTables, aggregations, or joins. Willa is just an experiment at this point, but hopefully it’s given you some food for thought. I’d also recommend checking out the `Noah` (TODO - link) library, that has similar goals but a slightly different approach. Thanks for reading, please don’t hesitate to email me if you have any questions or comments.
