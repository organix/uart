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

## Object/Environment/Dictionary

In order to break away from positional access to data
(or linear offset-based relative-addressing),
we use logical names to access values 
within a particular environment.
The environment for execution of an actor's behavior
is rooted at the Event dispatched to "animate" the actor.
The Event structure could be something like this:

    {
        "sponsor" : { ...sponsor properties and resources... },
        "target" : {
        	"behavior" : [ ...a sequence of instructions... ],
        	...additional stateful actor properties...
        },
        "message" : ...the value sent...
    }

### Instructions

Notice that the value associated with `"behavior"` 
is an array (sequence) of instructions.

#### Actor Creation

    {
        "prototype" : Create,
        "behavior" : [ ...a sequence of instructions... ]
    }

#### Asynchronous Messaging

    {
        "prototype" : Send,
        "target" : anActor,
        "message" : ...the value to be sent...
    }

#### Changing Behavior

    {
        "prototype" : Become,
        "behavior" : [ ...a sequence of instructions... ]
    }

#### Binding Names

    {
        "prototype" : Bind,
        "name" : ...variable name...,
        "expr" : { ...an expression... }
    }

#### Object Values

    {
        "prototype" : Object,
        "name" : ...variable name...
    }

    {
    	"prototype" : Load,
    	"object" : ...object to read...,
    	"key" : ...namespace key...,
        "name" : ...variable name...
    }

    {
    	"prototype" : Store,
    	"object" : ...object to update...,
    	"key" : ...variable name...,
    	"value" : ...the value to assign...
    }

## Abstract Assembly Language

We don't want to be tied to any particular assembly language,
so we define an abstract assembly language 
in terms of basic operations on actor-visible resources.
Translating this abstract assembly language 
into assembly (or machine instructions) for a particular CPU
should be straight-forward.
[LLVM](http://llvm.org/) is our immediate target for translation.

### Instruction Descriptions

In the following descriptions, 
`reg` indicates a string which names a logical "register"
whose value may be read/written by the instruction.
Literal values are limited to strings, numbers, `true`, `false`, and `null`.

    { "action": "literal", "value": value, "result": reg }
    { "action": "object", "result": reg }
    { "action": "load", "object": reg, "key": reg, "result": reg }
    { "action": "store", "object": reg, "key": reg }
    { "action": "length", "object": reg, "result": reg }
    { "action": "split", "object": reg, "at": reg, "head": reg, "tail", reg }
    { "action": "concat", "head": reg, "tail": reg, "result": reg }
    { "action": "compare", "this": reg, "that": reg,
        "equal": reg, "less":reg, "more":reg }
    { "action": "add", "this": reg, "that": reg,
        "result": reg, "overflow":reg, "zero":reg, "pos":reg, "neg":reg }
    { "action": "sub", "this": reg, "that": reg,
        "result": reg, "underflow":reg, "zero":reg, "pos":reg, "neg":reg }
    { "action": "mul", "this": reg, "that": reg,
        "result": reg, "overflow":reg, "zero":reg, "pos":reg, "neg":reg }
    { "action": "div", "this": reg, "that": reg,
        "result": reg, "modulus":reg, "zero":reg, "pos":reg, "neg":reg }
    { "action": "if", "condition": reg, "true": [...], "false": [...] }
    { "action": "create", "behavior": reg, "result": reg }
    { "action": "send", "target": reg, "message": reg }
    { "action": "become", "behavior": reg }

Actor addresses behave like other values, 
except that there is no "literal" format.
They can only obtained through `"create"`,
although they can be included in messages
and other objects.

### Instruction Format

Behaviors are described in terms of 
imperative machine instructions.
Machine instructions consists of 
a _verb_ describing the operation, 
a _subject_ the verb acts on, 
an _object_ parameterizing the action, 
and a _result_ location.
The full instruction format is:

    verb(subject, object) => result

Not all components are present in all instructions.
Components may be either literal values 
or variable references.
Literal values (constants) are indicated by a `#` prefix.
Variable references are simply names.
We assume the assembler can do liveness analysis
and assign names to machine registers.

The instruction format shown above 
can be considered "syntactic sugar" 
for construction of a typed object such as:

    {
        prototype: MachineInstruction,
        verb: verb,
        subject: subject,
        object: object,
        result: result
    }

### Values

Our abstract machine has one primitive data-type,
a fixed-width string of binary digits 
that can be interpreted as an unsigned 
or signed (2's complement) integer ring.
We do not specify the exact width. _[maybe provide with bit-width as a constant]_
It is intended to match the natural size 
of a machine word on the target processor.
This will be either 16 or 32 bits
on most processors currently in use.

### Names

Rarely can a behavior be described using only constants.
Most behaviors require some kind of parameterization.
The lambda-calculus substitution model 
gives us a mechanism to describe parameterized behaviors.
We can define template functions 
that are applied to parameter values.
The resulting substitution produces a concrete behavior.
We use sets and maps, rather than lists, 
to define the parameters and their substitution values.
For example:

    \{m, a}.[
    	#send(m, a)
    	#send(m, a)
    ] => b
    b{a:sink, m:#0}

Here we define a template function 
that introduces the set of names `m` and `a` 
as parameters to a block of instructions. 
This function is bound to the name `b`,
then applied to a map of parameter values.
The result is effectively equivalent to:

    #send(#0, sink)
    #send(#0, sink)

In this context, the name `sink` is expected to be bound
in some enclosing scope.  An implementation may use 
partial application in the compilation process 
to substitute concrete values 
for some parameter names.

The same binding mechanism is used 
to provide access to the incoming message. 
The message is treated as a parameter value 
applied to the behavior template 
to produce the concrete behavior of an actor.
_[is there an implicit binding to `this` or `self`?]_

### Memory Management

Construction begins with allocation of unstructured storage.

    #alloc(size) => address

The `#alloc` verb allocates 
`size` consecutive words unstructured storage.
The result `address` is a reference to this memory.

Unreferenced storage is automatically reclaimed 
by a garbage collection process.
Sometimes we can assist the garbage collector
by explicitly releasing storage.

    #free(address)

The `#free` verb releases the storage at `address`
and invalidates further use of that reference.

Transferring data between registers and memory involves
specifying a bit-position in memory, 
a direction for the transfer, 
and a count of the number of bits to copy.

	#from(base, offset)

The `#from` verb establishes the bit-position 
for subsequent `#read` operations.
If `base` is zero, 
the relative `offset` is added to the current read-position.
If `base` is non-zero, 
the absolute read-position is set to `base` words plus `offset` bits.

    #read(count) => data

The `#read` verb copies `count` bits 
from the current read-position in memory 
into the least-significant bits of a `data` register,
and increments the current read-position by `count`.

	#to(base, offset)

The `#to` verb establishes the bit-position 
for subsequent `#write` operations.
If `base` is zero, 
the relative `offset` is added to the current write-position.
If `base` is non-zero, 
the absolute write-position is set to `base` words plus `offset` bits.

    #write(data, count)

The `#write` verb copies the least-significant `count` bits
from a `data` register
into memory at the current write-position,
and increments the current write-position by `count`.

	#copy(data) => register

The `#copy` verb copies `data` to a `register`.
Note that `data` may be either a literal value or a register,
but `register` must be a register reference.

### Actor Behavior

The minimal actor behavior 
is to ignore the incoming message 
and do nothing.
When a behavior is complete, 
control returns to the kernel
to dispatch the next event.

    #done

The `#done` verb indicates that the actor
has handled the event. _[successfully?]_

An actor is created with some initial behavior.

    #create(behavior) => address

The `#create` verb creates a new actor
with the specified initial `behavior`.
The result is the unique `address`
of the new actor.

Asynchronous message sending
is the most common kind of actor behavior.

    #send(message, target)

The `#send` verb creates a new event
that will asynchronously deliver 
the specified `message` 
to the `target` actor.

Sometimes an actor's behavior
is dependent on the history
of messages it has processed.

    #become(behavior)

The `become` verb specifies a new `behavior`
that the current actor will use
to process future message events.

## Bit-Stream Transport

Communication between memory domains 
is accomplished through reading and writing bit-streams.
[Cap'n Proto](http://capnproto.org) is our model for message encoding.
_[This implies that our abstract machine word-size is 64 bits.]_

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

Three bits are sufficient to encode a wide range of bit-string sizes.

    CODE  WIDTH
    000   0 bits    (empty)
    001   1 bit
    010   2 bits
    011   4 bits
    100   8 bits    (1 byte)
    101   16 bits   (2 bytes)
	110   32 bits   (4 bytes)
    111   64 bits   (8 bytes)
