---
layout: documentation
title: Publish-Subscribe with RethinkDB
active: docs
docs_active: publish-subscribe
permalink: docs/publish-subscribe/python/
switcher: true
language: Python
---

The
[publish-subscribe pattern](http://en.wikipedia.org/wiki/Publish-subscribe)
is a powerful way to decouple applications that need to
communicate. RethinkDB [changefeeds](/docs/changefeeds) allow us to
implement publish-subscribe with the database acting as a message
exchange. We've built a small example library called
[repubsub](https://github.com/rethinkdb/example-pubsub/tree/master/python)
implementing the pattern for use in Python applications.

This article will explain how to use repubsub, as well as describe how
it's implemented on top of changefeeds. If your application needs
asynchronous broadcast notifications, this may be a good fit.

# Publish-Subscribe #

There are different publish-subscribe variations, so here we'll
describe the type using a central topic exchange. In this model,
publishers connect to the central exchange and broadcast a message
with a given topic. When subscribers connect, they notify the exchange
about what kinds of messages they're interested in. The exchange is
then responsible for filtering messages.

Becaused of this decoupled interaction, publishers are free to
disconnect whenever they want. There may even be more than one
publisher. Likewise, if no subscribers are currently listening for
messages with a certain topic, the exchange is free to simply delete
them.

# Using repubsub #

Repubsub implements a simple abstraction on top of RethinkDB to enable
publish-subscribe. It uses ReQL as the filtering mechanism, so the
full power of the language is at your disposal. This gives a lot more
flexibility than traditional message queues.

The repubsub library has three classes:

* An `Exchange` is created by both publishers and
  subscribers. Publishers put messages into the exchange, and
  subscribers listen to messages on the exchange.
* A `Topic` is used by publishers. It contains some key that contains
  meta-data about the messages.
* A `Queue` is used by consumers. It has two purposes:
   1. To buffer messages that the subscriber hasn't consumed yet (this
      buffering is actually done in the database server)
   2. To filter messages from the `Exchange` by their `Topic` (again,
      the server does this filtering)

To import repubsub and create a connection to an exchange:

```python
import rethinkdb as r
import repubsub

connection = r.connect(db='repubsub')
exchange = repubsub.Exchange(connection, name='pubsub_demo')
```

Note that `connection` is just a normal RethinkDB connection.

## Subscribing to topics using regex ##

The simplest case is publishing a message with a string for a
topic. This lends itself to using regexes for filtering.

To publish a message to the exchange, create a topic:

```python
topic = exchange.topic('fights.superheroes.batman')
```

Now we can publish any arbitrary JSON document to the topic:

```python
topic.publish({
    'opponent': 'Joker',
    'victory': True,
})
```

In the subscribing application we need to create a queue to receive
and buffer messages. The queue takes a filtering function as an
argument. This is similar to what you would pass to
[filter](/api/python/filter). Here we'll subscribe to all messages
about superhero fights:

```python
filter_func = lambda topic: topic.match(r'fights\.superheroes.*')
queue = exchange.queue(filter_func)
```

Then, to listen to messages, just iterate over the `.subscribe()`
method on the queue:

```python
for topic, payload in queue.subscribe():
    print 'I got the topic:', topic
    print 'With the message:', payload
```

## Subscribing to topics using tags ##

You can also filter messages by tags. We could put the tags into a
string and build a regex to match messages with the tags we want, but
luckily we have the full power of ReQl at our disposal. Instead, we
can make the topic an actual JSON array, and use ReQL's
[contains](/api/python/contains) method to do the filtering.

So, for example, if we wanted to send a notification that Batman and
the Joker had a fight:

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

In this case, we would only receive notifications of fights involving
a superhero. Fights between supervillains would be ignored.

## Subscribing to hierarchical topics ##

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


## Trying out the repubsub demo ##

The example documentation includes a
[demo script](https://github.com/rethinkdb/example-pubsub/blob/master/python/demo.py')
that shows off the three topic patterns described above. The script
implements both a publisher and a subscriber with each pattern
type. You can use this script to try out multiple publishers and
multiple subscribers to test out how it works.

Run the publisher and correspoding subscribers in different terminal
windows, so the output doesn't run together. For example, to run the
publisher for the regex demo:

```bash
$ ./demo.py regex publish
```

and in another window run:

```bash
$ ./demo.py regex subscribe
```

You can run the `tags` and `hierarchy` demos the same way.

# How the library is implemented #

As mentioned above, the repubsub library is built using RethinkDB
changefeeds. Briefly, here's how it works:

* Each exchange is a single RethinkDB table
* Each document in the table has 4 keys: `id`, `topic`, `payload`, and
  `_force_change`.
    * For every message sent, repubsub sets the `_force_change` key
      randomly
    * RethinkDB doesn't generate a change notification if a document
      is updated to an identical value, so we force a change.
* When posting a message to a topic, first repubsub attempts to
  overwrite a document with the exact same topic. If the exact topic
  isn't found, it creates a new document with the topic.
* Subscribers create a changefeed on the `Exchange`'s table, filtering
  for changes that mention documents matching their topic queries.

The entire changefeed query on the exchange is:

```python
# self.table is the Exchange's underlying table
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

While repubsub is easy to use, it doesn't come with any testing. It's
intended as an example of using publish-subscribe and if you intend to
use it in production, it's a good idea to do extensive testing first
to make sure it works for your use case.

Also, users should be aware that RethinkDB provides no guaranteed
delivery for change notifications. Because of that, repubsub is not a
good choice to build a task queue where lost tasks are a serious
issue. Many applications, (for example: push notifications to users),
aren't as sensitive to lost messages, and repubsub is a better choice.
