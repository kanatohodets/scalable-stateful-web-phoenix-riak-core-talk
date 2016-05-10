# Stateful web apps with phoenix and riak core

Let's build a 

* stateful
* distributed
* riak core
* fault tolerant
* phoenix
* real time
* impress-your-cat

application!

# Stateful and stateless: a short history of web application architecture
* stateful: the web servers store data that persists between requests. like
  putting a user's working data set into a hash in your web app code.
* stateless: they don't.

(picture of a stateless architecture).

Going to gamble that _most_ web applications are designed this way; this has
been the case since CGI. Why? What about all that memory on our web servers?

Latency numbers -> it would be awesome to use memory more! Even memory + some
in-DC network trips would be pretty great!

So why do we almost always use a stateless architecture?

* statelessness makes it easy to scale horizontally by adding more web servers.
  load balancing is a trivial problem (round robin, pick at random, etc.)
* the nature of the web workload: a stateless request/response protocol (HTTP)
* the scope of state in memory is very limited:
    * applications often written using short-lived programs that handle
        1 request at a time (Python, Ruby, Perl, PHP). memory has a short lifespan. hit a wall for reuse immediately
    * when using long-lived daemons and event loops, memory to a single thread.
      hit a wall for reuse when scale exceeds one thread
    * difficult to coordinate between application servers without a broker:
      memory limited to a single server.
      hit a wall for memory re-use as soon as scope of work exceeds 1 box

Ok, that makes sense for my Python/Ruby/Perl/PHP application. How about
Phoenix? 

Horizontal scalability? Well, that one's a bit mixed: you get horizontal
scalability at the web level, but pay for it by pushing everything down to the
database, where you get a new round of scalability headaches. Still, applies to
a Phoenix app as much as a Python/Ruby/Perl/PHP one.

scope of memory?
* long-lived multi-threaded BEAM process, so the memory on one app server
  can be used over and over
* coordination between application servers -> distributed elixir!

# Distributed

Great! Let's start stashing user data in ETS tables on our Phoenix app! If that
user makes a request that routes to a different box, just send a message to the
place where the data is stored. Awesome!

## Suddenly, distributed systems!

Coordinating state between distinct services put us firmly into the realm of
distributed systems. "distributed systems" sounds scary and hard, but really
it's just about coordinating multiple servers: in our case, about coordinating
some data that represents state in our web application. 

good thing we're using elixir! I'll just `send(remote_pid, :give_me_the_data_please)`!

wait! distributed elixir is a basic building block, not a pattern, not a framework. 

When it comes to coordinating distributed state, this:

    send(remote_pid, {:get_user_data, user_id})

... or even this:

    GenServer.call(remote_pid, {:get_user_data, user_id})

Are primitives, not patterns! They're the 'print' statement of distributed
systems.

Rolling your own multi-node state coordination using ad-hoc message passing is
like writing web apps like this:

    <?php
        $foo = mysql_query($my_cool_query)
        echo "<div>$foo</div>"
    ?>

... and sometimes that's the right tool! we've all written code like that. but
web development recognized this as a painful way to write larger applications,
and adopted patterns, like MVC. So where are our patterns in distributed
systems, and which ones can we use to help us build some stateful Phoenix apps?

### Not CAP

may have heard about "CAP": Consistency, Availability, Partition tolerance.

People talk a _lot_ about CAP when discussing distributed systems. It'd be easy
to think of it as something like a 'major paradigm' for designing distributed
systems, right?

A quick (and sloppy) definition in case you've never bumped into this TLA: 

* Consistency: If I ask two servers in my system the same question, will they
  tell me the same thing? (as opposed to a stale value)
* Availability: will a request to a non-failing node in my system result in
  a successful response?
* Partition tolerance: How well can I tolerate network partitions and server
  failures?

The general notion is that you can "pick two" to have in your system, but you
can't have all 3.

We can discuss the precise semantics of these terms, but pretty much the first
thing you notice is that this is _super_ unhelpful for actually designing
a system. And that's ok! CAP is not a pattern like MVC; it's shorthand for
talking about the tradeoffs available in distributed systems designs. What are
some of those designs?

#### CP
* consensus algorithms (get everyone to agree on some value)
* multi-master synchronous replication (every write gets copied to every other
  node before it is acknowledged) 
* ... others

#### CA
* a tricky, and disputed area. some say it doesn't exist in any useful form, others disagree. in terms of web apps, probably safe to say it isn't applicable, because partitions pretty much always always happen (http://i.imgur.com/tO4TgCj.jpg). 

#### AP
* gossip (psst, node A has user 42's data!)
* distributed hash tables (sha1("user 42") --> server A)
* CRDTs (Phoenix Presence)

### So which one of these helps us write a cool stateful, distributed web application?
It depends! But if you're interested in stateful web apps in part because you
want to remove or reduce the bottleneck on the database, and your data won't
fit in a single server's RAM, you probably want some kind of AP system. But
don't get too hung up on "AP" or "CP" -- real software tends to use
combinations of techniques from both buckets.

Speaking of real software...

Just like we'd rather not write a templating engine for every web app we write,
it would be much nicer to reuse some existing implementation of these
distributed systems patterns...

# Introducing... Riak Core

Riak Core is part-framework, part-toolkit for writing distributed systems that
follow the Dynamo pattern.

## Dynamo?
A paper published by Amazon in 2007 about a collection of distributed systems
techniques (gossip, consistent hashing, others) that they used to create
a highly available key-value store. There's a lot in there, but for our
purposes of writing a stateful, distributed web application, the most important
aspect is the _the ring_.

## The Ring
I watched a lot of Basho conference talks to prepare for this one, and every
single one had this slide: (ring slide). Maybe putting this slide here means
I get to go work at Basho?

But for the sake of diversity, let's do something different. Everyone knows and
loves plain old hash tables, so let's talk about those.

They're backed by an array with some number of buckets. 

    [[], [], [], []]

When you do something like this:

    my_hash_table["some key"] = "handy value"

The runtime hashes "some key" to turn it into a number, like 33, then does 33
modulo `$number_of_buckets` (which is 4). The result decides _which_ slot in the hash table
to put "handy value" into. In this case, 1. So we get:

    [["handy value"], [], [], []]

And so next time we go looking for "some key" we know where to find "handy
value".

Plus some tricks if we put too much stuff into the table and we need to make the underlying array longer.

(diagram)

So, distributed hash tables work on the same principle: the only difference is
that instead of a single array underlying our hash, we assign responsibility
for different buckets to different servers. (diagram)

And that's pretty much the main concept in a Riak Core application! Plus some
tricks for changing which nodes are responsible for a particular set of
buckets. Except in Riak Core land, we use the word "vnode" instead of "bucket".

So, anything we can hash, we can distribute! Great! Let's build a really simple
riak core application: the pingring. This will hash the current time, then send
the `:ping` request to the appropriate server.

# The parts of a basic `riak_core` app:

* service -> API for interacting with the ring.
* vnode -> rubber -> road. represents a set of buckets in the hash table.
  implements the `vnode` behaviour, has callbacks to handle commands, as well
  as a bunch of ring maintenance stuff.
* supervisor -> starts up the vnode master
* app -> starts up the supervisor and registers the service with riak core


So what do we need in order to ping?

service code: 
vnode: (`handle_command`, etc.)
supervisor: start `vnode_master`
app: start sup, register vnode, register service 

So, let's run it!

https://github.com/kanatohodets/elixir_riak_core_ping

here's a 3 node cluster running. I'll ping in one of the consoles. (pong, pong,
pong). Cool! we can see that the hashed timestamp gets directed to different
locations on the ring.

So now we have a sketch of an elixir application running on `riak_core` that
distributes some CPU work. That's already pretty slick: if you're interested in
some kind of worker cluster to distribute tasks, this is a pretty handy looking
pattern already. But! We've been talking about stateful web applications, so
let's manage some state!

We'll write a simple, totally non-production-ready in-memory KV store using ETS.

So, the first approach is pretty simple: add two new `handle_command` clauses
to the vnode -- one for store, and one for fetch. We'll add the same APIs to
our service module, and presto!

https://github.com/kanatohodets/elixir_riak_core_broken_kv

Unfortunately, this is probably a little bit too simple. The problem is that
our ping and image store only write to _one_ vnode `get_primary_apl(doc_idx,
1)`. This means that a single node failure will leave us without some of our
data, which is a real bummer.

# Fault Tolerant

I promised a fault tolerant application, so this is where we'll do it.

Basically, we need to write our data to more than one vnode -- exactly how many
is called the `n` value -- and wait for acknowledgement from some of them (the
`w` value) that the data have been written. When we read, we can _also_ request
to read from multiple vnodes (the `r` value), in case one of them is
unavailable right now, or we're concerned about mismatches.

This is an area where you need to do a bit more legwork: `riak_core` does not
provide a generalized approach to solve this problem.

How does this work? Well, never fear: we'll write a FSM. This FSM will
'coordinate' operations with some replication factor, send out the requests,
and then enter a 'waiting' state until it hears back from a sufficient number
of vnodes that the command has been executed, at which point will message our
'service' process with the answer.

Some code: (op FSM supervisor, op FSM basics, service 'receive')

https://github.com/kanatohodets/phoenix-ricor-kv/tree/master/apps/picor

Slick, a `riak_core` application in Elixir. How about Phoenix?

# Phoenix

So we have two applications: a phoenix web app, and a riak core ring app, and
we want them to interoperate. Umbrellas to the rescue! Umbrella applications
are perfect for this. It's incredibly easy for Phoenix controllers to use the
Service API in our `riak_core` ring application. Let's look!

https://github.com/kanatohodets/phoenix-ricor-kv

We'll add a route for pinging, for store, and for fetch.

Then demo each one. Nice! That was easy! Now we have a super awesome
stateful, distributed, Riak Core, fault tolerant, Phoenix application! How cool
is this?

# Realtime

So now we've looked at a fairly standard usage of `riak_core`: as a simple K/V
store. What about something a little more interesting? Something cool with
channels?

Well... 

The basic characteristic of riak core is a distributed hash table. In order to
distribute work around the hash ring, we need a key to hash on. Do channels
have anything like that? Yes! Topic! Every message sent through a channel
(including subscribe/unsubscribe) includes the topic that identifies that
stream of communication: things like "room:lobby" or "game:42". This is the
_perfect_ thing to hash!

So! `Phoenix.PubSub.Ricor`: hash channel messages on their topic and have
a vnode responsible for managing the subscriber list/broadcasting
messages. This works! Let's look! https://github.com/kanatohodets/hashpub

What could you do with this? Well, right now the vnode just manages the
subscriber list, but you could imagine it doing something neat, like storing
history for that topic. Or even more neat, like managing the FSM for game state
for the players of a particular match, identified by topic.

## Great, I want to build one!

soon! `mix ricor.new --phoenix --opfs`, potentially also `GenVnode` to provide
default callbacks and wrap some of the `riak_core` macros.

## Scalable == awesome (--> suitable)

So now that we've talked about _how_ you might build such an application, the
real trick is determining _when_ we should use this approach. Garrett Smith
gave a great talk about how when we say "scalable" we often mean "awesome" --
it's a poorly defined word that gets used to convey enthusiasm about cool
technology. This talk is firmly in that vein: I have not done this in
production: in fact, I'm a total amateur in the BEAM world!

Garrett proposes that instead of "scalable" we should aspire for our software
to be "suitable" -- fit for purpose, that meets the needs of our users.

So, with that in mind, despite the hefty dose of "the joy of cool tech", I hope
that some folks might look at this powerful toolkit and find some inspiration
for excellent (and suitable :)) software they could write using it.

Some ideas where this sort of thing might be suitable are: services with _many_
small topics. Infrastructures with fairly memory-heavy hardware going to waste
a bit on app servers.

Drawbacks: hardcore distributed system, no more SQL queries, much harder to
reason about the state/health of the system.

As for building "impress your cat" applications, well...

(grumpy cat is never impressed)
(with apologies to Garrett)

# `{:exit, :talk_over}` 

Many, many thanks to Mariano Guerra, Mark Allen, and Ryan Zezeski: their
wonderful talks, learning materials, example applications, and scaffolding for
Riak Core applications in Erlang made this talk possible. 
