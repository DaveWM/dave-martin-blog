+++
date = 2019-11-19T00:00:00Z
description = "A walkthrough of how to update a cardinality many attribute in Datomic in Clojure"
draft = true
title = "Datomic how-to: Update Cardinality Many Attribute"

+++
I've recently encountered a slight difficulty with updating [cardinality many](https://docs.datomic.com/on-prem/schema.html#cardinality "Datomic cardinality documentation") attributes in Datomic, and I thought I would make a short walkthrough post about it, to help out anyone else who runs into the same issue.

## The Problem

In Datomic, each attribute has a "cardinality", signifying how many values an attribute is allowed. Most attributes have cardinality one, but they can also have cardinality many, meaning they can have multiple values. _Adding_ values for a cardinality many attribute is fairly straightforward, but updating them to a fixed set is more difficult. Let's take the example of a simple Todo list app, where each todo has a title and multiple tags. Our schema looks like this:

```clojure
(def schema
  [{:db/ident :todo/title
    :db/valueType :db.type/string
    :db/cardinality :db.cardinality/one}
   {:db/ident :todo/tags
    :db/valueType :db.type/string
    :db/cardinality :db.cardinality/many}])
```

Creating a new todo with a list of tags is fairly straightforward:

    (d/transact
     conn
     [{:todo/title "Do the dishes"
       :todo/tags ["whenever"]}])

However, updating the tags for a todo is more difficult. It's easy to _add_ a value for the attribute, but not so easy to update them. Let's say we want to update the tags to be `["important" "today"]`, how would we do that?

## The Solution

One solution is to:

1. Query the current values of the attribute
2. Diff them with the new values
3. Use that diff to create [transactions](https://docs.datomic.com/on-prem/transactions.html "Datomic transactions documentation")
4. Apply the transactions using `d/transact`

Here's the code to do this:

    (defn update-attr-txs [db entity-id attr values]
      (let [;; Step 1
            current-vals (-> (d/q '[:find [?p ...]
                                    :where [?id :intention/parents ?p]
                                    :in $ ?id]
                                  db
                                  entity-id)
                             (set))
            ;; Step 2
            [added removed] (clojure.data/diff (set values) current-vals)
            ]
        ;; Step 3
        (concat (->> added
                     (map #(-> [:db/add entity-id attr %])))
                (->> removed
                     (map #(-> [:db/retract entity-id attr %]))))))
     
     ;; Step 4
     (->> (update-attr-txs (d/db conn) 1 :todo/tags #{"important" "today"})
         (d/transact conn))

That should be all you need! I've created a gist containing the `update-attr-txs` function [here](https://gist.github.com/DaveWM/66bced07550aaf295a3f40dbf263f171 "gist"). You can find some alternative solutions in [this StackOverflow post](https://stackoverflow.com/questions/39432061/updating-value-with-cardinality-many "Related StackOverflow post"), and also [this one](https://stackoverflow.com/questions/42112557/datomic-schema-for-a-to-many-relationship-with-a-reset-operation "another StackOverflow post").

I hope this helps you out, please don't hesitate to get in touch if you have any questions.