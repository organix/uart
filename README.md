# uArt - micro actor run-time

uArt attempts to provide a minimal run-time environment for actors.
The goal is to create a bridge between the Actor Model of computation
and the realities of modern computer hardware architecture.
In particular, we assume a CPU (possibly multi-core) accessing data 
and executing instructions from a linearly addressable memory.

The primitive actions defined by the Actor Model are:
 * _Create_ a new actor with some initial behavior
 * _Send_ an asynchronous message to an actor
 * _Become_ a new behavior for processing subsequent messages


## Design

Resource control is an important issue in an actor implementation.
The model abstracts away resource limitations, 
but a practical implementation must deal with this issue.
Our approach to resource control involves the concept of 
a _Sponsor_ for an actor computation.

We observe that all actor computation occurs 
in the context of receiving an _Event_.
It seems logical the the Sponsor for a computation 
should either **be** the Event or be **reachable through** the Event.
Thus, the Event provides the context for a computation.

Each Create action produces a new uniquely identifiable actor.
We use a distinct memory address to identify each actor, 
assuming a memory-safe computational environment 
to maintain the security of each actor.
Of course, the implementation kernel 
has privileged access to all actors.
Kernel code is "meta" to the actor semantics, 
and should be kept to an absolute minimum.

At a minimum, an actor consists of a sequence of CPU instructions
representing the actor's behavior.
The actor's identity corresponds to the memory address
where their behavior starts.
Thus, dispatching an Event to an actor is accomplished by 
preparing an environment and simply jumping to the behavior.
We can avoid using a call-stack by making direct jumps 
rather a "subroutine" call and return.
An actor behavior executes as an "atomic" transaction.
The behavior indicates completion 
by simply jumping back to the kernel.
We expect to have separate jump addresses 
to distinguish between successful completion 
and signaling some kind of exception.

The execution environment for an actor's behavior 
consists of at least these critical components:
 * The unique Event being delivered and processed
 * The unique identity of the target actor
 * The code constituting the Behavior of the actor
 * The data constituting the State (if any) parameterizing the Behavior
 * The data constituting the Message to be delivered
 * The Sponsor that controls resources available during processing

Note that these may, in fact, all be components of a composite Event.

    Event
    +-> Sponsor
    +-> Actor
    |   +-- Behavior
    +-- Message

Each Send action produces a new uniquely identifiable Event
with a specified Message and a target actor address.
The same actor may (of course) be the target of several Events,
thus the actor must be _referenced_, not _contained_, by the Event.
The Sponsor is also likely to be shared, and therefore referenced.

An important distinction can be made between actors (as described above)
and value-actors.  Value-actors, or simply _Values_, do not have
distinct identities.  They represent immutable values.  They may be
duplicated and/or locally recreated when crossing memory domains.
This is an important optimization because it avoids having to route messages 
back the "the original" in cases where the behavior of the local value
is indistinguishable from any/all other copies.  Values are still actors,
in the sense that they can receive messages.  However, 
they cannot use the _Become_ primitive, so their behavior never changes.

Within a memory domain, 
only one actor can occupy a particular address.
This implies that we can determine matching actor identities 
by simply comparing memory addresses.
We can extend this to a small set of pre-defined Values 
by encoding them directly in the bits of a "pointer".
These are effectively synthetic actor addresses.
The type of the value (and thus it's immutable behavior) 
and the value itself are both encoded in the address.

The main resource controlled by the _Sponsor_ is memory.
Since each Send and each Create consumes memory, 
an actor cannot get very much work done 
without support from a Sponsor.
We anticipate some sort of "watch-dog" timer
to prevent run-away CPU consumption,
but expect most behaviors to be well-behaved 
by virtue of having been created by a trusted compiler
and working within a resource-safe language.

## Bit-Stream Transport

Communication between memory domains 
is accomplished through reading and writing bit-streams.
[Cap'n Proto](http://capnproto.org) is our model for message encoding.

### Encoding Examples

    struct NullableBool {
        union {  // 16-bit type-tag in bits 0-15
            null @0 : Void;
            bool @0 : Bool;
        }
    }

    16#0000000100000000 ------------0000 = Null
    16#0000000100000000 --------00000001 = False
    16#0000000100000000 --------00010001 = True

    CODE  WIDTH
    000   0 bits    (empty)
    001   1 bit
    010   2 bits
    011   4 bits
    100   8 bits    (1 byte)
    101   16 bits   (2 bytes)
	110   32 bits   (4 bytes)
    111   64 bits   (8 bytes)
