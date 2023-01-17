+++
date = 2023-01-16T00:00:00Z
description = "An exploration of real-time reactive web apps using Clojure, Datomic, and websockets"
draft = true
title = "The Web Before Teatime"

+++
It's increasingly common for web applications to incorporate real-time elements. More and more, users expect the page to be updated instantly, without the need to refresh. Unfortunately, this is often implemented as an afterthought. The dominant paradigm for the web is still request-response. This is well suited to fetching data on some client-side event (e.g. page loads, button clicks), but not for receiving real-time updates from the server. To work around this limitation, developers often resort to polling, or "sprinkle on" ad-hoc real-time events using a websocket. This inevitably leads to a buggy, inconsistent app that's difficult to work on. It's also common for different parts of the page to update at different frequencies. Perhaps some parts are completely static, some update on a fixed interval, and others update in real time. This has been a problem since websockets were first implemented in browsers, back in 2011. Surely now we can do better?

One potential way forward is outlined in Nikita Prokopov's (a.k.a. Tonsky) seminal blog post ["The Web After Tomorrow"](https://tonsky.me/blog/the-web-after-tomorrow/ "The Web After Tomorrow"). It was written in 2015, but is still highly prescient today. In it, Tonsky outlines a potential architecture for real-time web apps. The basic idea is that clients send queries to the backend, and then the query results are streamed back as a series of deltas (for example Datomic's [datoms](https://docs.datomic.com/cloud/whatis/data-model.html "Datomic Datoms")). The front end uses these deltas to populate its own "database" of application state, and then renders the page from that. This is a brilliant idea, if only it could be fully realised. Using this hypothetical framework you would only have to design your database schema, craft queries for the frontend data, then write the view layer. If this could be achieved in a reliable and performant way, it would be a game changer.

There have been a few attempts to create this architecture over the past few years. One of these is the [DatSync](https://github.com/metasoarous/datsync "DatSync") collection of libraries. Another is the [3DF](https://github.com/sixthnormal/clj-3df "3DF") client and library. However, both of these projects are unfinished and development appears to have stalled. Javascript's [Meteor](https://www.meteor.com/ "Meteor") solves a subset of the problem, but it doesn't get anywhere near a complete solution. On the database side, there's very interesting work being done on databases like [Materialize](https://materialize.com/ "Materialize DB"), [ksqlDB](https://ksqldb.io/ "ksqlDB") and [RethinkDB](https://rethinkdb.com/ "RethinkDB"), which offer streaming query results. However, none of them use a suitable query language - you can't allow your frontend to execute arbitrary SQL statements. Also, they are all still fairly immature technologies.

This all started me wondering - how close can we get to _The Web After Tomorrow_, today? Although there are still large missing pieces, I wanted to see if there was any practical way of working around them. The solution didn't have to be perfect, just an improvement on today's standard of ad-hoc websocket messages.

I decided to try building a simple real-time rock-paper-scissors web app, getting as close to the "Web After Tomorrow" architecture, using technologies available today. Datomic has lots of cool features (history, forking, filtering) that aren't available in other DBs, so I wanted to use that. I wasn't aiming for a massively scalable architecture, but whatever I came up with had to perform reasonably well.

### Missing Pieces

I quickly determined that there are 3 major missing pieces:

#### #1 - Streaming Queries

Ideally, we want to be able to pass an arbitrary query to the database and have results stream back. When the database is updated in a way that affects the query, we want to know the new result. Unfortunately, very few databases offer this. The solution mentioned in _The Web After Tomorrow_ is to create a "reversible" query language - one that can be used to query the database, but also to determine if a database transaction affects the query results. Although there has been [some progress](https://materialize.com/docs/sql/subscribe/) on this front, it's sadly missing from all mainstream databases. Datomic allows you to monitor transactions using the [transaction report queue](https://blog.datomic.com/2013/10/the-transaction-report-queue.html), but there's no way to determine if a transaction affects an arbitrary query.

#### #2 - Authorisation

In any non-trivial web app, users only have permission to view a subset of the database. Even in something as simple as a todo app, users can't be allowed to query each other's todos. Query results must therefore be restricted somehow. In traditional web apps, this is done manually at the endpoint level. As far as I'm aware, there aren't any conventions around doing this for arbitrary database queries. Datomic's [database filters](https://docs.datomic.com/on-prem/time/filters.html "Datomic Database Filters") could potentially be used, but encapsulating all your authentication logic in a single predicate is difficult in practice.

#### #3 - Consistency

One other major problem is guaranteeing consistency on the client. How do we guarantee that the client doesn't miss deltas from the backend, in the face of potential network dropouts and server failures? Missing even a single delta could be disasterous, and lead to incorrect data being shown to the user. We also can't rely on the browser being refreshed to fix this. We need a guarantee that all deltas are received at least once, and in the correct order. Unfortunately, to my knowledge there are no existing protocols or libraries that help us here.

### Workarounds

In the face of these missing pieces, it's currently impossible to achieve the full "Web After Tomorrow" architecture. However, I found that with a couple of concessions you can get most of the way there.

#### Workaround #1 - Sending Full Query Results

One decision I made was to send full query results to the client, rather than datoms or deltas. There are a few reasons for this. The main reason is that it allows you to sidestep the "Consistency" problem mentioned above. The frontend's state now only depends on the latest query result, so you can be sure it's always correct (albeit perhaps out of date).

Secondly, sending the first query result as a delta is a bit awkward. When the client first sends a query, it needs to know the whole query result. To do this you have to run the query, get the full result, then convert it to a sequence of deltas. While this is possible, it's a tad inelegant.

The third reason is that the client may need to be sent some data that is calculated on-the-fly on the server. Doing this as deltas is possible, but you must then draw a slightly awkward distinction between "DB" deltas and "calculated" deltas.

The format of the deltas is also a pain point. Using [datoms](https://docs.datomic.com/cloud/whatis/data-model.html) is natural, but this essentially forces the client to use ClojureScript and [datascript](https://github.com/tonsky/datascript). There are other formats, like [JSON Patch](https://jsonpatch.com/), but these each have their own problems and tradeoffs.

#### Workaround #2 - Introducing a Query DSL

I also ended up introducing an application-specific DSL for queries, as opposed to arbitrary datalog. The frontend sends queries in this DSL, and then the backend translates it into a datalog query which it passes to Datomic. Since queries are now far more constrained, the backend can more easily determine which transactions affect which running queries. For my app, I found a tuple of `[:query-type id]` sufficed for this query DSL. I called queries written in this DSL "subscriptions", to distinguish them from database queries.

One other advantage of having our own DSL is that subscriptions can be crafted to minimise the amount of unnecessary data sent to the client. Ideally, each subscription should return data that changes together, and at a similar rate. This allows us to minimise the inefficiency of transmitting the entire query result on each update.

This constrained query language also makes authorisation far easier. For example, let's say you're writing a todo app and have a subscription like `[:todo 1234]`. In this case, it's trivial to determine whether the todo `1234` belongs to the current user.

### The End Result

You can check out the app I ended up with at [rock-paper-scissors.live](https://rock-paper-scissors.live). It looks like this:

![](/rps-demo.gif)

The source code for the backend is [here](https://github.com/DaveWM/rps-backend), and the frontend [here](https://github.com/DaveWM/rps-frontend).

I've also created templates you can use to create your own app. There's [one for the frontend](https://github.com/DaveWM/real-time-frontend), and [another for the backend](https://github.com/DaveWM/real-time-backend). Just follow the instructions in the readmes to get started.

The basic architecture I ended up with is broadly similar to the "Web After Tomorrow" architecture, but with several key differences. The front end sends a subscription to the backend, which records it in an atom. As discussed previously, this subscription is written in a DSL rather than datalog. I used a basic tuple of `[:query-type entity-id]`, like `[:game 1234]` or `[:player "Bob"]`. When a query is first sent, the backend immediately queries the DB, and pushes the entire query result back down to the client. So far, so easy...

Now for the tricky bit! Within the backend, there is a thread that is responsible for reacting to database transactions. It does this by monitoring Datomic's [transaction report queue](https://blog.datomic.com/2013/10/the-transaction-report-queue.html). This "transaction watcher" determines which subscriptions need to be updated, re-runs the queries for these subscriptions, then pushes the results to the subscribed clients. Since this logic has to be hand-coded, it unfortunately requires a fair bit of manual work. The flip side is it allows you make lots of assumptions to improve performance. For example, in my app I made the assumption that a single transaction could only affect one game.

The front end is responsible for managing its own subscriptions (e.g. unsubscribing when the user navigates off a page), but subscriptions are automatically removed when the client disconnects.

The resulting code for handling subscriptions looks like this:

```clojure
(when (subs/authorised? sub user-id) ;; 1
  (let [db (subs/init-sub! sub user-id db) ;; 2
        result (subs/fetch-sub sub db)] ;; 3
    (swap! sub->users* update sub conj user-id) ;; 4
    (reply-fn {:data (subs/format-for-user sub result user-id) ;; 5
               :sub  sub})))
```

1. Check whether the user is allowed to subscribe.
2. Initialise the subscription, for example create the entity being subscribed to if needed.
3. Convert the subscription to a datalog query, and use it to query the DB
4. Record the subscription in the sub->users atom
5. Format the query result for the user, and send it to the frontend

The transaction watcher looks something like this:

```clojure
(Thread.
  #(while true
     (let [{:keys [db-after tx-data] :as evt} (.take (db/tx-queue)) ;; 1
           affected-subs (subs/affected-subs 
                           (keys @sub->users*) 
                           evt 
                           tx-data)] ;; 2
       (doseq [sub affected-subs]
         (let [users (get @sub->users* sub)
               result (subs/fetch-sub sub db-after)] ;; 3
           (doseq [user users]
             (server/chsk-send! ;; 4
               user
               [:server/push {:data (subs/format-for-user sub result user)
                              :sub  sub}])))))))
```

1. Listen for transactions, using Datomic's [transaction report queue](https://docs.datomic.com/on-prem/transactions/transaction-processing.html#monitoring-transactions "Datomic Transaction Report Queue")
2. Determine which current subscriptions are affected by the transaction
3. For each affected subscription, query the database
4. For each user for these subscriptions, format the result for the user and send it over the websocket

There are, of course, some drawbacks to this architecture. The primary disadvantage is the amount of manual work involved, which can possibly lead to errors and performance problems. The backend needs to be taught how to convert each query type to a datalog query, and also how to determine which queries a transaction affects. When writing this code you have to be vigilant about performance, particularly in the transaction watcher. A naive transaction watcher that queries the DB per transaction per subscription will be unacceptably slow.

### Conclusion

Unfortunately, it appears that we still have some way to go before we get to _The Web After Tomorrow_. There are several critical missing pieces, and progress appears to have slowed down recently. However, it is still possible to write a real-time app with a (fairly) minimal amount of effort. If you're looking to create your own app, I hope the templates I've provided are of some use - or at least give you some inspiration. If you have any comments or feedback, please feel free to [email me](mailto:mail@davemartin.me). Thanks for reading!

 

#### Links

* [The Web After Tomorrow](https://tonsky.me/blog/the-web-after-tomorrow/)
* Templates
  * [Front End](https://github.com/DaveWM/real-time-frontend)
  * [Back End](https://github.com/DaveWM/real-time-backend)
* Rock Paper Scissors example app
  * [App](https://rock-paper-scissors.live)
  * [Backend source code](https://github.com/DaveWM/rps-backend)
  * [Frontend source code](https://github.com/DaveWM/rps-frontend)