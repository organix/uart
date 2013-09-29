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
should either *be* the Event or be *reachable through* the Event.
Thus, the Event provides the context for a computation.

Each Create action produces a new uniquly identifiable actor.
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
We expect to have separate kernel addresses 
to distinguish between successful completion 
and signaling some kind of exception.

The execution environment for an actor's behavior 
consists of at least these critical components:
 * The unique Event being delivered and processed
 * The Sponsor that controls resources available during processing
 * The data constituting the Message to be delivered
 * The code constituting the Behavior of the actor
 * The data constituting the State (if any) parameterizing the Behavior
 * The unique identity of the actor
Note that these may, in fact, all be components of a composite Event.
```
    Event
    +-- Sponsor
    +-- Message
    +-- Actor
        +-- Behavior
```
