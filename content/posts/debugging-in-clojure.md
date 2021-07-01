+++
date = 2021-06-30T23:00:00Z
description = "A guide to debugging in Clojure, using the REPL"
draft = true
title = "Debugging in Clojure"

+++
Whenever I speak to other Clojure developers, whether they're seasoned pros or brand new to the language, one question that always comes up is "how do you debug your code?". Most of us have heard the rhetoric around REPL driven development - "develop your program interactively in the REPL, debugging as you go!". This sounds straightforward, but it's often not that easy in practice. Say you're writing a REST API using Compojure, and you notice one of the endpoints is returning an incorrect response. How do you go about debugging it? In this case it's not as easy as "construct a map in the repl, pass it to your function, and see if the output is what you expect". You may have made some incorrect assumptions about the request map, or a function within your route handler is returning something unexpected. Mutable state also gets in the way - things like database connections, Kafka consumers, and HTTP servers. In this guide, I'll detail my personal approach debugging in situations like this.

When I first find a bug, the first thing I do is some good ol' fashioned println debugging. It may be primitive, but it's often the quickest way to diagnose the problem! In other languages, it can be a bit awkward trying to shoehorn `console.log` statements into your code. Luckily, Clojure's reader tags come in handy here. I'm a big fan of the spyscope library, which exploits reader tags to full effect. It's dead simple - once you've installed it as per the readme, just put #spy/p in front of the form you want to print out. It's my go-to library for easily pretty-printing values in the REPL. When that code is executed, it'll print the result of the form to the REPL, with nice colours to boot! This is great for quickly seeing what a form evaluates to, which is often all you need to diagnose the problem.

For cases when you need to debug a piece of code more thoroughly, the scope-capture library is indispensable. It allows you to capture the local environment whenever a certain form is evaluated, and then re-create this environment in the REPL. This allows you to play around in the REPL as if you'd stopped the program at the point when the form was evaluated. This works really well with Stuart Halloway's "divide and conquer" approach to debugging. To set it up, just follow the guide in their readme, then wrap the form you want to debug in the "sc.api/spy" macro. The next time the form is evaluated, you'll see a message like "SPY \[1 -1\] ..." in your REPL. Now you can run "(sc.api/defsc 1)" in the REPL to recreate the captured environment. You can then evaluate forms within the spy, and see what the result would have been.

One debugging trick I've found very useful is to set up a command in your editor for def'ing let bindings. This helps both when debugging pure functions, and using the scope-capture approach detailed above. Let's say you have a function like this: 

```clojure
(defn get-owner-and-pets [owner-id] 
  (let [owner (db/get-owner owner-id) 
        pet-ids (:pet-ids owners) 
        pets (->> pet-ids (map db/get-pet))]
    (assoc owner :pets pets))) 
```

Imagine it's not working as expected. One approach to debugging it would be to def the owner-id in your REPL and then evaluate each of the let bindings in turn. You would run "(def owner-id 1234)", then evaluate the let bindings one-by-one into your REPL, like "(def owner (db/get-owner owner-id))". This can be a bit laborious when you have a big let binding, and I've found setting up a command in your editor is well worth the effort. If you're using IntelliJ and Cursive, you can add the following as a custom REPL command (TODO: link): 
```clojure
(def ~selection) 
```
Now just select the code inside the square brackets in the let statement, and then run your custom command. All the let bindings should be evaluated and def'd in your REPL.

There are other some tools that I've played around with, but personally found I don't use heavily. REBL, Reveal, and Portal are graphical tools for viewing Clojure data structures. They're a great idea, but to be honest I tried out REBL and found I didn't use it much in practice. In the course of researching for this blog post I came across Debux, a library which provides some nice debugging utilities. I haven't had a chance to properly evaluate it yet, but it looks like a very helpful library. If you're using Cursive, you can also use Intellij's built-in debugger. This may be useful in some circumstances, but in general I've found that using scope-capture is easier and more effective. The debugger pauses the thread which is executing the code, which can often cause havok with other threads (in a Kafka Streams topology for example).

I hope that gives you a few useful tips for debugging Clojure code. If you want to read more on the subject, there are lots more excellent guides to debugging \[TODO links\] here. I'd also highly recommend reading "REPL Debugging: No Stacktrace Required". Thanks for reading!