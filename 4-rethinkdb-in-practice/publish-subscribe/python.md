---
layout: documentation
title: Publish-Subscribe with RethinkDB
active: docs
docs_active: publish-subscribe
permalink: docs/python/publish-subscribe/
switcher: true
language: Python
---

The publish-subscribe pattern is a powerful way to decouple
applications that need to communicate. RethinkDB
[changefeeds](/docs/changefeeds) allow us to implement
publish-subscribe with the database acting as a message
exchange. We've built a
[small example library](https://github.com/rethinkdb/example-message-queue/tree/master/python)
implementing the pattern for use in Python applications. If your
application needs sub

This article will explain how to use the example library as well as
how it's implemented on with changefeeds.

# Publish-Subscribe #

There are many different variations on publish-subscribe, so here
we'll just describe a topology using a central topic exchange. In this
model, publishers connect to the central exchange and broadcast a
message with a given topic. When subscribers connect, they notify the
exchange which kinds of messages they're interested in. The exchange
is then responsible for doing filtering of messages.

Becaused of this decoupled interaction, publishers are free to
disconnect whenever they want. There may even be more than one
publisher. Likewise, if no subscribers are currently listening for
messages with a certain topic, the exchange is free to simply delete
them.

# Using the library #

The example library implements a very simple abstraction on top of
RethinkDB to enable publish-subscribe. It uses ReQL as the filtering
mechanism, so the full power of the language is at your disposal. This
gives a lot of flexibility when designing routing patterns.

To import the library and create a connection to an exchange:

```python
import rethinkdb as r
import repubsub

connection = r.connect(db='repubsub')
exchange = repubsub.Exchange(connection, name='pubsub_demo')
```

Note that `connection` is just a normal RethinkDB connection.

## Regex topics ##

The simplest case is publishing a message with a string for a
topic. This lends itself to using regexes for filtering.

To publish a message to the exchange, create a topic:

```python
topic = exchange.topic('weather.ca.mountainview')
```

Now we can publish any arbitrary JSON document on the topic:

```python
topic.publish({
    'temp': 70,
    'cloudy': False,
})
```

In the subscribing application, we need to create a queue. The queue
takes a filtering function as an argument. This is similar to what you
would pass to [filter](/api/python/filter).

```python
filter_func = lambda topic: topic.match(r'.*ca.*')
queue = exchange.queue(filter_func)
```

Then, to listen to messages, just iterate over the `.subscribe()`
method on the queue:

```python
for topic, payload in queue.subscribe():
    print 'I got the topic:', topic
    print 'With the message:', payload
```

## Array of tag topics ##

Another method of routing might be based on the topic being a list of
tags. We could put the tags into a string and building a complicated
regex to match messages with the tags we want, but luckily we have the
full power of ReQl at our disposal. Instead, we can make the topic an
actual JSON array, and use ReQL's [contains](/api/python/contains)
method to do the filtering.

So, for example, if we wanted to send a notification that a fight took
place between Batman and the Joker, we might do:

```python
topic = exchange.topic(['superhero', 'fight', 'supervillain'])
topic.publish({
    'interaction_type': 'tussle',
    'participants': ['Batman', 'Joker'],
})
```

Then, subscribers could listen for messages with any combination of tags:

```python
filter_func = lambda tags: tags.contains('fight', 'superhero')

for tags, payload in exchange.queue(filter_func).subscribe():
    fighter1, fighter2 = payload['participants']
    print fighter1, 'got in a fight with', fighter2
```

In this case, only notifications of fights involving a superhero would
be received. Fights between supervillains would be ignored completely.

## Hierarchical object topics ##

As a final example, we'll use an object as the topic. Using an object
as the topic allows us a richer hierarchical structure, rather than
keeping them in a flat structure like an array. This provides us with
maximum flexibility in message routing.

Let's say we want to publish the teaming up between Batman, Superman
and the Joker:

```python
topic = exchange.topic({
    'teamup': {
        'superheroes': ['Batman', 'Superman'],
        'supervillains': ['Joker'],
    },
    'surprising': True
})

topic.publish('Today Batman, Superman and the Joker teamed up '
              'in a suprising turn of events...')
```

There are multiple subscriptions we could have set up that would receive this news:

```python
# Get all surprising messages
surprising_filter = lambda topic: topic['surprising']

# Get all messages involving a teamup or a fight
teamup_filter = lambda topic: topic['teamup'] | topic['fight']

# Get all messages talking about a teamup with Batman
batman_query = lambda topic: topic['teamup']['superheroes'].contains('Batman')
```


## Demo ##

Included in the example implementation is a demo script that shows off
the three topic patterns above. The script implements both a publisher
and a subscriber with each pattern type. You can use this script to
try out multiple publishers and multiple subscribers to test out how it works.

Run the publisher and correspoding subscribers in different terminal
windows, so the output doesn't run together. For examepl, to run the
publisher for the regex demo:

```bash
$ ./demo.py regex publish
```

and in another window run:

```bash
$ ./demo.py regex subscribe
```

You can run the `tags` and `hierarchy` demos the same way.

You may also want to have a look at the [source code](https://github.com/rethinkdb/example-message-queue/blob/master/python/demo.py')
of the demo.

# Implementation of the library #

As mentioned above, the implementation of the pub-sub library builds
on RethinkDB changefeeds. Briefly, here's how it works:

* Each exchange is a single RethinkDB table
* Each document in the table has 4 keys: `id`, `topic`, `payload`, and
  `_force_change`.
    * For every message sent, the `_force_change` key is set randomly.
    * No notification will be sent if a document is updated to an
      identical value, so this ensures a change notification is
      always generated.
* When posting a message to a topic, first the library attempts to
  overwrite a document with the exact same topic. If the exact topic
  isn't found, it creates a new document with the topic.
* Subscribers create a changefeed on the Exchange's table, filtering
  for changes that mention documents matching their topic queries.
* Each time a change is generated, the library pulls out the `topic`
  and `payload` fields and passes them to the user.

The entire changefeed query on the exchange is:

```python
# self.table is the Exchange's table
# filter_func is the function passed in by the subscriber
self.table.changes()['new_val'].filter(lambda row: filter_func(row['topic']))
```

This query pulls out `new_val` from the changefeed, and passes just
the topic field from the new value down to the subscriber's function.

```python
for message in self.full_query(binding).run(self.conn):
    yield message['topic'], message['payload']
```

## Caveats ##

While this library is easy to use, it doesn't come with any
testing. It's intended as an example of using publish-subscribe and if
you intend to use it in production, it'd be a good idea to do
extensive testing first.

Also, users should be aware that change notifications can be lost,
there's no guaranteed delivery for messages. Because of that, this
library is not a good choice to build a job queue. Any applications
using the library should behave well in the presence of missed
messages.
