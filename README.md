# Itc4net: interval tree clocks for .NET

Itc4net is a C# implementation of Interval Tree Clocks (ITC), a causality tracking mechanism. 

*Disclaimer: While this project is intended for production use, it has not yet been used in production. Towards this goal, there is an extensive set of unit tests, but it still requires real-world use and testing.*

### Overview

This project is a C#/.NET implementation of the ideas presented in the 2008 paper, [Interval Tree Clocks: A Logical Clock for Dynamic Systems](http://gsd.di.uminho.pt/members/cbm/ps/itc2008.pdf). An Interval Tree Clock (ITC) provides a means of causality tracking in a distributed system with a dynamic number of participants and offers a way to determine the partial ordering of events.

The term *causality* in distributed systems originates from a concept in physics where "causal connections gives us the only ordering of events that all observers will agree on" ([The Speed of Light is NOT About Light](https://youtu.be/msVuCEs8Ydo?t=44s) | [PBS Digital Studios | Space Time](https://www.youtube.com/channel/UC7_gcs09iThXybpVgjHZ_7g)). In distributed systems, physical clocks are problematic because of drift, synchronization issues, leap seconds, and double-counting—just to name a few (see, [the trouble with timestamps](https://aphyr.com/posts/299-the-trouble-with-timestamps) for more details). In short, there is no global clock. A causal history (or compressed representation) is necessary to determine the partial ordering of events or detect inconsistent data replicas because physical clocks are unreliable.

It's worth mentioning, in this context, a causal relationship implies one thing could have potentially influenced another, such as, change A was known before change B occurred.

### Getting Started

To get started, the best way to become familiar with the basics of interval tree clocks is to read the [ITC paper](http://gsd.di.uminho.pt/members/cbm/ps/itc2008.pdf). The first 3 sections provide an overview and explain the fork-event-join model. It is helpful, but not necessary to understand how the kernel operations are implemented.

Install using the NuGet Package Manager:

```
Install-Package Itc4net -Pre
```

#### Stamp Constructor

An ITC stamp is composed of two parts: an ID and event tree (i.e., casual past). The Stamp default constructor creates a *seed* stamp.

```c#
Stamp seed = new Stamp(); // (1,0)
```

Itc4net uses immutable types for the core ITC Stamp, Id, and Event classes; therefore, the ITC kernel operations (fork, join, and event) are pure functions. As a consequence, it is important to treat a Stamp like a struct type and replace the Stamp variable reference with a new reference when performing any kernel operations. 

For example, do this:

```c#
Stamp s = new Stamp();	// (1,0)
s = s.Event();			// (1,1) inflated event
s = s.Event();			// (1,2) inflated event
```

Do **not** do this, as it will lead to incorrect results:

```c#
Stamp s = new Stamp();	// (1,0)
s.Event();				// (1,1) inflated event
s.Event();				// (1,1) oops, same result as before!
```

#### Fork

The fork operation generates a pair of stamps with distinct IDs, each with a copy of the event tree. This operation is used to generate new stamps (after creating the initial seed stamp). More specifically, it splits the ID of the original stamp into two distinct IDs, so it is important to keep a reference to one stamp and share the other. In the next example, variable *s* would likely be a private field stored in a class responsible for generating timestamps.

```c#
Stamp s = new Stamp();

Tuple<Stamp, Stamp> forked = s.Fork();
s = forked.Item1; // replace variable s with one of the new stamps
Stamp another = forked.Item2; // a logical clock for use by another process
```

Since extracting multiple items from tuples is awkwardly verbose, there are extension methods that provide an alternative API with out parameters, if preferred:

```c#
Stamp s = new Stamp();
Stamp another;
s.Fork(out s, out another);
```

Multiple stamps can be generated by successive calls to Fork. Although it may not be common (in practice) to create multiple stamps at once, it is useful enough that there are extensions to generate 3 or 4 stamps at a time.

```c#
Stamp s = new Stamp();		// (1,0)
var forked = s.Fork4();
Stamp s1 = forked.Item1;    // (((1,0),0),0)
Stamp s2 = forked.Item2;    // (((0,1),0),0)
Stamp s3 = forked.Item3;    // ((0,(1,0)),0)
Stamp s4 = forked.Item4;    // ((0,(0,1)),0)
```

or, alternatively & more compactly, using out parameters:

```c#
Stamp s = new Stamp();
Stamp s1, s2, s3, s4;
s.Fork(out s1, out s2, out s3, out s4);
```

**[New]** Using C#7 tuple deconstruction allows an even more convenient syntax:

```c#
Stamp s = new Stamp();		// (1,0)
(Stamp s1, Stamp s2, Stamp s3, Stamp s4) = s.Fork4();
```

*A note about logical clock identifiers: the ID of a logical clock needs to be unique in the system, one for each participant. Some approaches use integers which works well when there is a global authority that can hand out identities or when a system uses a fixed number of participants. Some approaches use UUIDs (or other globally unique naming strategies) which allows any number of participants, but tracking casual history for each participant leads to very large timestamps. Instead, ITC uses the fork operation to generate a distinct pair of IDs from an existing stamp. This allows a dynamic number of participants and eliminates the need for a global authority, as any (non-anonymous) stamp can generate new IDs.*

#### Peek

The Peek operation provides an anonymous stamp with a copy of the event tree (causal past). An anonymous stamp has an ID equal to zero, so it cannot inflate the event tree. Its purpose is to provide a stamp that can be stored with a data record or passed with a message.

```c#
Stamp _s; // Private member, e.g., (((1,0),0),(1,1,0))

Stamp GetTimestamp()
{
	_s = _s.Event();	// Inflate stamp (((1,0),0),(1,(1,1,0),0))
  	return _s.Peek();	// return anonymous stamp
}

void SendMessage(byte[] data)
{
  	Stamp timestamp = GetTimestamp(); // e.g., (0,(1,(1,1,0),0)) <-- the ID is always 0
  	
  	MessageBody message = new MessageBody(data);
	message.Headers["Timestamp"] = timestamp.ToString();
	_bus.Send(message);
}
```

#### Event

The Event operation inflates the stamp which is roughly equivalent to ticking of the logical clock. 

```c#
Stamp s = new Stamp();	// (1,0)
s = s.Event();			// (1,1)
```

*Note: An ID is necessary to inflate a stamp, since the inflation occurs within the area of the event tree that belongs to the ID. That is, the Event method ticks its own clock and leaves causal information for other IDs unchanged. Invoking the Event method on an anonymous stamp will succeed; however, it will return an unchanged anonymous stamp since it cannot inflate the event tree.*

#### Join

The Join operation merges two stamps into one, merging the ID and event trees (causal past). The ID and Event trees are normalized, if possible.

```c#
Stamp s1 = Stamp.Parse("(((1,0),0),(0,(1,1,0),0))");
Stamp s2 = Stamp.Parse("(((0,1),0),(0,(1,0,1),0))");
    
s1 = s1.Join(s2); // ((1,0),(0,2,0))
s1.ToString().Dump();
```

Illustrating with ITC graphical notation:

![join-example-diagram](docs/img/join-example-diagram.png?raw=true)

Joining stamps with IDs is useful when a participant leaves the system (and no longer needed) because it essentially retires the ID and has the potential to reduce the size of the stamp, as illustrated in the example above. Joining with an anonymous stamp (ID = 0) will simply merge the event tree, a behavior that is used by the Receive extension method (below).

#### Send

The Send extension method performs an atomic event & peek, which is commonly used to capture causal information to pass with a message. 

You'll notice this example is functionally equivalent to the Peek operation example. 

```c#
Stamp _s; // Private member, e.g., (((1,0),0),(1,1,0))

void SendMessage(byte[] data)
{
  	Stamp timestamp;
  	_s = _s.Send(out timestamp); // (0,(1,(1,1,0),0))
  	
  	MessageBody message = new MessageBody(data);
	message.Headers["Timestamp"] = timestamp.ToString();
	_bus.Send(message);
}
```

Illustrating with ITC graphical notation:

![send-example-diagram](docs/img/send-example-diagram.png?raw=true)

*Note: Stamps are immutable and the stamp is inflated as part of Send, so the method must return two stamps. The return value is the inflated stamp (with ID) and the out parameter is the the inflated anonymous stamp.*

#### Receive

The Receive extension method performs an atomic join & event, which is commonly used to merge causal information from a received message with the receiver's causal information.

```c#
Stamp _s; // Private member, e.g., ((0,(1,0)),(0,0,(0,2,0)))

void ReceiveMessage(MessageBody message)
{
  	Stamp timestamp = message.Headers["Timestamp"]; // (0,(1,(1,1,0),0))
  	_s = _s.Receive(timestamp); // join + event = ((0,(1,0)),(1,(1,1,0),(0,2,0)))
  	
  	// ... process the message ...
}
```

Illustrating with ITC graphical notation:

![receive-example-diagram](docs/img/receive-example-diagram.png?raw=true)

#### Other Examples

The Itc4net.Tests project includes [tests that demonstrate different scenarios](https://github.com/fallin/Itc4net/blob/master/src/Itc4net.Tests/ScenarioTests.cs) which may be useful as more complete examples, including:

- Example workflow from ITC 2008 paper, section 5.1
- Ordering of events example (vector clocks)
- Example from Detection of Mutual Inconsistency in Distributed Systems, Fig. 1 (version vectors)

### Some Things to Consider

Interval Tree Clocks are a fascinating approach to causality tracking, but ultimately the partial ordering of events (vector clocks) or the detection of inconsistent data replicas (version vectors) are combined with many other techniques within a distributed system. For example, [Amazon's Dynamo](http://s3.amazonaws.com/AllThingsDistributed/sosp/amazon-dynamo-sosp2007.pdf) paper combines many techniques, including: [consistent hashing](https://github.com/fallin/Wildling/blob/master/Wildling.Core/PartitionedConsistentHash.cs), versioning, membership, gossip protocols, failure detection, and more, to form a cohesive system. With causality tracking, like everything in computer science, there are no one-size-fits-all answers, so here are some things to consider:

Obtaining or distributing stamps among multiple participants requires careful consideration:

- The stamp creation workflow involves creating a seed stamp and then using the fork operation to generate stamps, with distinct IDs, for each participant. Any process/participant requiring its own source of causal history will require its own stamp with a distinct ID. Who creates the seed stamp? Is it created by the initial leader of the cluster or does an administrator create the seed and allocate the initial stamps during initial setup?
- How does a newly started process/participant acquire a stamp? The fork operation generates new stamps, but presumably a process will need to perform a remote method call (or send a message) requesting an existing peer to perform the fork operation and share a new stamp.
- What happens when a participant (e.g., service process) restarts? It is not desirable to abandon a stamp because forking new stamps increases the size of all future stamps, not to mention losing the process's casual past. 
  - Should a process persist the ITC stamp so it can recover state and its stamp when restarting? How often should this be done? How much causal history is it acceptable to lose? If a process maintains local storage so it can recover its state when restarting, then consider storing the ITC stamp as well. 
  - Alternatively, if a process does not use persistent storage and must bootstrap from other participants when restarting, then consider notifying other participants, during graceful shutdown, so they can perform a join operation to retire the ID.
- When permanently removing a participant, consider notifying other participants in the system. When a stamp contains an ID, the join operation can be used to retire the ID (sum & normalize the ID trees).

Threading & Synchronization Concerns

- The core ITC classes (stamp, ID, and event) are immutable so they are inherently thread-safe. However, it transfers synchronization concerns to the developer using the library. For example, imagine a process (e.g., ASP.NET WebAPI application) that maintains a single ITC stamp to track causal history for the entire process. While the core operations themselves are thread-safe, any operations on the shared Stamp variable instance need to be synchronized since the variable needs to be written between each operation. That is, concurrent calls to Event are not the same as serialized calls to Event.
  - [Future Enhancement?] It may be useful to provide a mutable wrapper class that performs synchronized operations on a shared ITC stamp. The wrapper class's API would be simpler because mutable versions of Event, Join, and Receive would not have to return a value and Send could simply return the anonymous stamp. It's unclear whether this would help or make the API surface more confusing (i.e., this is immutable, but this other class is not!)
  - Another option is to use a Stamp for each thread.

### Additional Resources

- [Interval Tree Clocks: A Logical Clock for Dynamic Systems](http://gsd.di.uminho.pt/members/cbm/ps/itc2008.pdf)
- [Logical time resources](https://gist.github.com/macintux/a79a254dd0bdd330702b): a convenient collection of links to logical time whitepapers and resources, including Lamport timestamps, version vectors, vector clocks, and more.
- [The trouble with timestamps](https://aphyr.com/posts/299-the-trouble-with-timestamps): the title says it all.
- [Provenance and causality in distributed systems](http://blog.jessitron.com/2016/09/provenance-and-causality-in-distributed.html): an excellent description of causality, why it's important and useful; a call to action!
- [Happened before and causal order discussion](http://cs.stackexchange.com/questions/40061/happened-before-and-causal-order): a helpful discussion on the meaning of causality.

### Implementation Details

- The 2008 ITC paper provides exceptional and detailed descriptions of how to implement the kernel operations (fork-event-join model). This project implements the timestamps, IDs, and events directly from the descriptions and formulas in the ITC paper; it is not a port of any existing ITC implementations.
- The core ITC classes are immutable. Accordingly, the core ITC kernel operations: fork, event, and join are pure functions.
- The ID and event classes use a [discriminated union technique](http://stackoverflow.com/a/3199453) (or as close as you can get in C#) to distinguish between internal (node) and external (leaf) tree nodes. A *Match* function eliminates the need to cast and provides concise code without the need for type checks and casting.
