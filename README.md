# Distributed Study Group on Distributed Systems
## TLDR:
We have a directed graph of sources and sinks. Sources emit streams of events that they send to sinks. For every source, each of its sinks must observe the source's stream in the same order in which the source generates the events. We'll build different implementations of a service satisfying this requirement. Our implementations will start with a simple Rails implementation -> Rails with system design optimizations -> simple Scala -> Scala with optimizations. We'll create a re-usable set of tools for profiling performance, stress and integration testing, comparing performance and failure points across all implementations.

## What this is:
A loose exploration of various concepts and tools in distributed systems by following Twitter's path from its creation with Ruby on Rails to its present custom stack built in-house for Scala. 

All the tools and practices currently employed in distributed systems exist as solutions to particular problems. Rather than picking up a hammer and looking for problems for which a hammer might be suited, better to start with an end goal in mind, build towards it, encounter problems, and let those problems that have naturally occurred motivate your search for a solution. Whenever we encounter a problem or want an abstraction, we will attempt to make our own (within reason, we will (probably) not be making our own databases). 

Some concepts to explore:
  + What are the important pitfalls and bottle-necks in scaling a service to greater load and better performance?
  + What are the some natural solutions to the problems encountered that we can build ourselves? What other solutions have been adopted by the community?
  + What are tradeoffs in the different options for deploying our services? (focus on existing solutions here: AWS, Heroku, Docker, Mesos)
  + Trade-offs in different datastores (CAP theorem + performance); performance improvement from different caching strategies, combination of datastores, load-balancing, etc. 
  + To what extent can we achieve performance improvements by changing system design while keeping language constant? 
  + To what extent can we achieve performance improvements by changing language/virtual machine while keeping system design constant? ("Moving to the JVM" is heard a lot - what payoff does the maturity and optimization of the JVM provide?)

Biggest picture system design question to think about is whether to pursue a client-server model (servers still running as part of a distributed system, but every event stream emitted by a source is mediated by some proprietary "Twitter server") vs a fully decentralized model in which nodes are all symmetric, and every source knows the address of its sinks, and sends its generated events directly to them without intermediation by some distinct Twitter server process. Not at all clear what the complexity of building or the performance characteristics of each would be; one clear difference in terms of failure points would be that the client-server model is much more exposed to complete service outages than is the decentralized system. If nodes are speaking directly to each other, the only possibility for the service to completely fail would be for extreme network partitions; some parts of the network may be cut off from each other, but every node will still be able to gather its input stream from some source. Another interesting difference would be how service discovery differs in the two designs. More to come along these lines.

===============
## Requirements

We'll start with some requirements for a system to be an acceptable implementation of Twitter.

Twitter is a graph of nodes joined in directed source-sink relationships. For nodes `x` and `y`, `(x -> y)` will denote the relationship in which `x` is the source and `y` a sink. 

For a given node `x`, `x.sources` denotes the collection of nodes from which `x` receives events. Each source has an event stream which it publishes to its sinks. Every sink has its own individual observed stream of events that is generated by merging together the emitted event streams of all the sources followed by the sink.

```
val sources: Set[Source] = x.sources
val inputStreams: Set[Stream[Event]] = sources.map(_.outputStream)
val combinedStream: Stream[Event] = inputStreams.reduce(mergeStreams)

// x.inputStream == combinedStream
```
So for every `x`, `x.sources` will generate a stream of events observed by `x`. A Twitter implementation is semantically correct if:
```
x.inputStreams.forAll(consistentWithSourceStream)
```
In words: 
  + Sources emit events in some sequence which they broadcast to sinks
  + In a correct Twitter implementation, for every source it must be the case that all of its sinks see the stream emitted by the source in the same sequence with which the source generates the stream.
  + In addition to all nodes receiving a stream consistent with a source's generated stream -- for every element of `Twitter.nodeSet.subsets` such that `subset.map(_.sources).reduce(_ intersect _).isNonEmpty`, all nodes in the subset must agree on the ordering of the stream formed by merging the output streams of all their shared sources. 
  + We will profile the performance of every Twitter implementation, seeing in which conditions our service will fail

## Concrete specification of the API:
We will start with a simple client program which makes requests against Twitter's API. The goal in this initial exercise is to identify a subset of Twitter's API that which all of our implementations will support. Details for this first step live in `/synthetic_twitter_feed`.

3) Simple Rails implementation:
  + Using as little additional configuration beyond the Rails defaults
  + Develop concrete API that will be used in all subsequent tasks
  + Develop library (scala or ruby) for semantic integration testing and performance profiling
  + Profile initial rails implementation

4) Optimize Rails implementation
  + Given info from integration and stress testing, identify bottle-necks in service and optimize
  + How much performance improvement is there in improving system design while keeping language constant?

5) Simple Scala implementation: 
  + Again, using as little additional configuration beyond the Play(?) defaults
  + Profile - identify bottlenecks and failure points

6) Optimized Scala implementation:
  + Given info from (5), change design as needed to improve performance
  + Profile: what sort of payoff from design changes vs change in language

=============




