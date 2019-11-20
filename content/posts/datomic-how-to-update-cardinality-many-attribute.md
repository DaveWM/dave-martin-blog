+++
date = 2019-11-19T00:00:00Z
description = "A walkthrough of how to update a cardinality many attribute in Datomic"
draft = true
title = "Datomic how-to: Update Cardinality Many Attribute"

+++
I've recently encountered a slight difficulty with updating cardinality many attributes in Datomic, and I thought I would make a short walkthrough post about it, to help out anyone else who runs into the same issue.

## The Problem

In Datomic, each attribute has a "cardinality", signifying how many values an attribute is allowed. Most attributes have cardinality one (`:cardinality/one`), but you are allowed to link multiple values to an attribute (`:cardinality/many`). Adding values for a `:cardinality/many` attribute is fairly straightforward, but updating them is more difficult. Let's take the example of a simple Todo list app, where each todo can have multiple tags. Our schema looks like this:

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

// TODO - code example of creating todo

However, updating the tags for a todo is more difficult. It's easy to _add_ a value for the attribute, but not so easy to update them. Let's say we want to update the tags to be "important" and "today", how would we do that?

## The Solution

The solution is to pull the current values of the attribute, diff them with what we want them to be, and use that diff to create transactions. Here's the code to do this:

// TODO - code to diff + create transactions

That should be all we need! I've created a gist with with the helper functions from the code above here (TODO: link). You can also find some alternative solutions in [this StackOverflow post](https://stackoverflow.com/questions/39432061/updating-value-with-cardinality-many "Related StackOverflow post"), and also [this one](https://stackoverflow.com/questions/42112557/datomic-schema-for-a-to-many-relationship-with-a-reset-operation "another StackOverflow post").

I hope this helps you out!