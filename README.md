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
by encoding them directly in the bits of a "register".
These are effectively synthetic actor addresses.
The type of the value (and thus it's immutable behavior) 
and the value itself are both encoded in the address.
The least-significant bits indicate the type of data stored.

    2#..xxxxx1  -- Integer ring
    2#..xxx000  -- String/Symbol
    2#..xxx010  -- Structure
    2#..xxx100  -- Reference
    2#..000100  -- null   (a pre-defined actor reference)
    2#..001100  -- true   (a pre-defined actor reference)
    2#..010100  -- false  (a pre-defined actor reference)
    2#..xxx110  -- Label/Address

The main resource controlled by the _Sponsor_ is memory.
Since each Send and each Create consumes memory, 
an actor cannot get very much work done 
without support from a Sponsor.
We anticipate some sort of "watch-dog" timer
to prevent run-away CPU consumption,
but expect most behaviors to be well-behaved 
by virtue of having been created by a trusted compiler
and working within a resource-safe language.


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
We assume the assembler can do liveness analysis
and assign logical registers to physical registers on a given CPU.

Registers may contain values or references.
The least-significant bits of the register
encodes the data-type, as described previously.

    { "action": "literal", "value": value, "result": reg }

The `"literal"` action stores a literal value 
into the register named by `"result"`.
Literal values are limited to numbers, strings, `true`, `false`, and `null`.
All other actions get their values indirectly, 
from registers named by strings, 
so this is the only way to provide an "immediate" parameter.

### Structures

Structures are collections of bindings from keys to values.
The keys can be of any type, although strings are most common.

    { "action": "struct", "result": reg }

The `"struct"` action creates a new empty structure value.
A reference to the new structure is stored 
into the register named by `"result"`.

    { "action": "load", "struct": reg, "key": reg, "result": reg }

The `"load"` action looks up `"key"` in `"struct"` 
and stores the corresponding value into `"result"`.
If `"key"` is not found in `"struct"`, then the value is `null`.
The prototype/nested-scope chain is included in the search.

    { "action": "store", "struct": reg, "key": reg, "value": reg }

The `"store"` action binds `"key"` to `"value"` directly in `"struct"`.
If value is `null`, then `"key"` is effectively deleted.

    { "action": "has", "struct": reg, "key": reg, "result": reg }

The `"has"` action stores `true` in `"result"` 
if `"struct"` has a direct binding for `"key"`.
Otherwise `false` is stored in `"result"`.
The prototype/nested-scope chain is not searched.

### Arrays

Arrays are a special kind of Structure.
They are index by numeric keys, starting with zero.
The value of the `"length"` key is always one larger
than the largest index under which a value is stored.
Thus, appending to the array is accomplished 
by storing the new element at the current value of `"length"`.

    { "action": "split", "value": reg, "at": reg, "head": reg, "tail": reg }

The `"split"` action create two new arrays from an existing array `"value"`.
The array stored in `"head"` contains the elements 
from the beginning of the array up to, but not including, `"at"`.
The array stored in `"tail"` contains the elements
starting from `"at"` and continuing through the end of the array.
Note that `"head"` and/or `"tail"` could be empty, 
depending on the value of `"at"`, 
and both have their own `"length"`.

The `"split"` action also acts on other types of `"value"`.
A string is interpreted as an array of integer code-points (for each character).
A number is interpreted as an array of boolean bit-values (most-significant first).

    { "action": "join", "head": reg, "tail": reg, "result": reg }

The `"join"` action concatenates two array values.
It is the inverse of `"split"`.
It also acts on strings and numbers.

### Arithmetic Operations

    { "action": "compare", "this": reg, "that": reg,
        "equal": reg, "less": reg, "more": reg }
    { "action": "add", "this": reg, "that": reg,
        "result": reg, "overflow": reg, "zero": reg, "pos": reg, "neg": reg }
    { "action": "sub", "this": reg, "that": reg,
        "result": reg, "underflow": reg, "zero": reg, "pos": reg, "neg": reg }
    { "action": "mul", "this": reg, "that": reg,
        "result": reg, "overflow": reg, "zero": reg, "pos": reg, "neg": reg }
    { "action": "div", "this": reg, "that": reg,
        "result": reg, "modulus": reg, "zero": reg, "pos": reg, "neg": reg }

### Actor Primitives

Actor addresses behave like other values, 
except that there is no "literal" format.
They can only obtained through `"create"`,
although they can be included in messages
and other structures.

    { "action": "create", "behavior": reg, "result": reg }
    { "action": "send", "target": reg, "message": reg }
    { "action": "become", "behavior": reg }

### Flow Control

Normally, instructions are executed in sequence.
The `"label"` action stores the address of the next instruction
into a named register.
Control can be redirected to a labelled location
by absolute or conditional jumps.

    { "action": "label", "name": label }
    { "action": "if", "condition": reg, "true": label, "false": label }
    { "action": "jump", "label" : label }

Several registers are pre-defined 
to contain components of the current Event.
This provides the execution environment
for the target actor's behavior.

 * `"_sponsor"`
 * `"_self"`
 * `"_message"`


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
