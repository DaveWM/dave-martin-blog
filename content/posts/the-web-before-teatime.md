+++
date = 2023-01-09T00:00:00Z
description = "An exploration of how to do real-time web apps using Clojure and Datomic"
draft = true
title = "The Web Before Teatime"

+++
It's increasingly common for web applications to incorporate real-time elements. More and more, users expect the page to be updated instantly, without the need to refresh. Unfortunately, these features are often implemented as an afterthought. The dominant paradigm for the web is still request-response. This is well suited to fetching data on some client-side event (e.g. page loads, button clicks), but not for receiving real-time updates from the server. To work around this limitation, developers often resort to polling, or "sprinkle on" ad-hoc real-time events using a websocket. This inevitably leads to a buggy, inconsistent app that's difficult to work on. It's also common for different parts of the page will update at different frequencies. Perhaps some parts are completely static, some update on a fixed interval, and others update from websocket messages. This has been a problem since websockets were first implemented in browsers, back in 2011. Surely over 10 years later we can do better?

One potential way forward is outlined in Nikita Prokopov's (a.k.a. Tonsky) seminal blog post "The Web After Tomorrow" (WAT). It was written in 2015, but is still highly prescient today. In it, Tonsky outlines a potential architecture for real-time web apps. The basic idea is that clients send queries to the backend, and then the query results are streamed back as a series of deltas (for example datomic's datoms). The front end uses these deltas to build up it's own "database" (e.g. re-frame's \`app-db\`), and then renders the page from that. This is a brilliant idea, if only it could be fully realised. Using this hypothetical framework you would only have to design your database schema, craft queries for the frontend data, then write the view layer. If this could be achieved in a reliable and performant way, it would be a game changer.

There have been a few attempts to create this architecture over the past few years. One of these is the DatSync collection of libraries. Another is the 3DF client and library. However, both of these projects are unfinished and development appears to have stalled. Javascript's [Meteor](https://www.meteor.com/ "Meteor") solves a subset of the problem, but it doesn't get anywhere near the full solution. It allows the front end to respond to changes on a collection or a single document, but not to an arbitrary query. On the database side, there's very interesting work being done on databases like Materialize, KSQL DB, and RethinkDB, which offer streaming query results. However, they are all still fairly immature.

This all started me wondering - how close can we get to "The Web After Tomorrow", today? Although there are still large missing pieces, I wanted to see if there was any practical way of working around them. The solution didn't have to be perfect, just an improvement on completely ad-hoc websocket messages. I decided to try building a simple real-time web app, getting as close to the WAT architecture, using technologies available today. Datomic has lots of cool features (history, forking, filtering) that aren't available in other DBs, so I wanted to use that. I wasn't aiming for a massively scalable architecture, but whatever I came up with had to perform reasonably well.

### Missing Pieces

I quickly determined that there are 3 major missing pieces:

#### #1 - Streaming Queries

Ideally, we want to be able to pass a query to the databaseThere's still no easy way of determining when query results change. The solution mentioned in "The Web After Tomorrow" is to create a "reversible" query language - one that can be used to query the database, but also to determine if a database transaction affects the query results. Although there has been some progress on this front, it's sadly missing from all mainstream databases.

#### #2 - Authorisation

In any non-trivial web app, users only have permission to view a subset of the database. Even in something as simple as a todo app, users can't be allowed to query each other's todos. Query results must therefore be restricted somehow. In traditional web apps, this is done manually at the endpoint level. As far as I'm aware, there aren't any conventions around doing this for arbitrary queries. Datomic's [database filters](https://docs.datomic.com/on-prem/time/filters.html "Datomic Database Filters") could potentially be used, but encapsulating all your authentication logic in a single predicate is difficult in practice.

#### #3 - Consistency

One other major problem is guaranteeing consistency on the client. How do we guarantee that the client doesn't miss deltas from the backend, in the face of potential network dropouts and server failures? We need a guarantee that all deltas are received at least once, and in the correct order. Unfortunately, that's easier said than done.

### Workarounds

In the face of these missing pieces, it's currently impossible to achieve the full "Web After Tomorrow" architecture. However, I found that with a couple of concessions you can get most of the way there.

#### Workaround #1 - Introducing a Query DSL

I ended up introducing an application-specific DSL for subscriptions, as opposed to arbitrary datalog queries. The frontend sends subscriptions in this DSL, and then the backend translates it into a datalog query so it can query the DB. It also allows the transaction watcher to determine which transactions affect which subscriptions. After researching the topic, I quickly realised that there is still no realistic way to do streaming datalog queries. I had high hopes for the 3DF library, but it's sadly still in alpha and hasn't had a commit since 2019. For the rock paper scissors app, I found a tuple of \`\[:subscription-type id\]\` sufficed for this subscription DSL. One other advantage of having our own DSL is that subscriptions can be crafted to minimise the amount of unnecessary data sent to the client. Ideally, each subscriptions should return data that changes together, and at a similar rate. This allows us to minimise the inefficiency of transmitting the entire query result on each update. Also, it makes authorisation much easier.

#### Workaround #2 - Sending Full Query Results

Another decision I made was to send full query results to the client, rather than datoms or deltas. There are a couple of reasons for this. The main reason is that sending query results as deltas requires that all messages reach the client, and that they arrive in order. This requires the aforementioned mechanism to guarantee this, which, as far as I could tell, nobody has built. Implementing a mechanism to do this is theoretically possible, but not practical. Secondly, sending the first query result as a delta is a bit awkward. When initially setting up a subscription, the client needs to know the whole query result. To do this, you have to run the query and then find the relevant datoms. While this is possible, it's a tad inelegant. The third reason is that the client may need to be sent some data that is calculated on-the-fly on the server. Doing this as datoms is possible, but you must then draw a slightly awkward distinction between "real" datoms and "client only" datoms. It's also worth considering that using datoms essentially forces the client to use ClojureScript and datascript. Also it's not ideal that database IDs leak out to the client. One disadvantage of sending the entire query result on each change is that it can result in a large amount of unnecessary data transfer. However, if you're careful about how the queries are split up, you can minimise this unnecessary traffic.

### The End Result

The basic architecture I ended up with is similar to the WAT architecture, but with several key differences. The front end sends a subscription to the backend, which records it in an atom. This subscription is not a datalog query, but instead a query DSL specific to your application. I used a basic tuple of \[:subscription-type entity-id\]. When a subscription is started, the backend immediately queries the DB, and pushes the entire query result back down to the client. Within the backend, there is a thread that is responsible for reacting to database transactions. It does this by monitoring Datomic's transaction report queue. This "transaction watcher" determines which subscriptions need to be updated, re-runs the queries for these subscriptions, then pushes the results to the subscribed clients. The front end is responsible for managing its own subscriptions, but subscriptions are automatically removed when the client disconnects.

There are, of course, some drawbacks to this architecture. The primary disadvantage is the amount of manual work involved, which leads to a possibility for error. The backend needs to be taught how to convert each subscription type to a datalog query, and also how to determine which subscriptions a transaction affects. When writing this code you have to be vigilant about performance, particularly in the transaction watcher. A naive transaction watcher that queries the DB per transaction per subscription will be unacceptably slow.

If all this sounds appealing to you, I've created a template here that you can use. You can also take inspiration from the Rock Paper Scissors reference example. If you'd like to see it in action, you can play a game of rock paper scissors here. Thanks for reading!