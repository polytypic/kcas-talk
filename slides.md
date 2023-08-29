---
title: Building a lock-free STM for OCaml
theme: black
revealOptions:
  transition: fade
  width: 1280
  height: 720
---

<!--
```ocaml
# #require "kcas_data"
# open Kcas
# open Kcas_data
```
-->

# Building a lock-free STM for OCaml

Vesa Karvonen

<small>Bartosz Modelski, Carine Morel, Thomas Leonard, KC Sivamarakrishnan, YSS
Narasimha Naidu, Sudha Parimala</small>

<img src="tarides.png" alt="Tarides">

note: Hello! I'm going to talk about a software transactional memory
implementation or library for OCaml that I've had the pleasure to work on with
the support of my colleagues.

---

▸ **_Introduction_**

Anatomy of a transaction

Scheduler agnosticism

Atomicity and Isolation

Performance

Conclusion

note: In this talk, I'm going to briefly highlight some features of the library,
illustrate some of its internal data structures and algorithms, discuss how the
library interoperates with other concurrent programming libraries, and more.

---

### Ingredients for Concurrent Programming

1. Mechanism to introduce independent threads
2. Mechanism for threads to communicate
3. Mechanism for threads to synchronize

note: If you open a textbook on concurrent programming, it will likely tell you
that you need three ingredients for concurrent programming.

---

### Transactional Memory

1. ~Mechanism to introduce independent threads~
2. Mechanism for threads to communicate
3. Mechanism for threads to synchronize

note: Transactional memory doesn't give you independent threads of control, but
it does give a nice mechanism for such threads to communicate and synchronize
with.

---

### Transactional Memory

- [Abstraction for shared memory concurrency](https://en.wikipedia.org/wiki/Transactional_memory)
- Databases
  - [Atomicity, ~Consistency~, Isolation, ~Durability~](https://en.wikipedia.org/wiki/ACID)
- Familiar
  - Just wrap imperative code inside transactions
  - Use traditional data structures

note: Transactional memory is basically an abstraction for shared memory
concurrency, where threads communicate via shared memory data structures. It is
inspired by the database world, where you have transactional databases allowing
you to manipulate state using queries with ACID properties. Perhaps the main
advantage of transactional memory is that it is, in some sense, familiar, as it
allows you to keep on using familiar looking sequential code and data
structures.

---

<a href="https://ocaml-multicore.github.io/kcas/"><img width="30%" src="kcas.svg"></a>

Kcas

<small>Software Transactional Memory for OCaml</small>

note: Our software transactional memory library is called Kcas.

---

### Contributions to main

<img src="contributions-to-main.png">

note: And as you can see from this graph, development of Kcas already started
many years ago.

---

### Kcas originally

- [Lock-free](https://dspace.mit.edu/handle/1721.1/73900) multi-word
  compare-and-set
  - aka MCAS or k-CAS
- Perform a list of CAS operations
- Low-level programming model
  - maintain state, retry loops, backoffs, ...
  - like with 1-CAS
- For implementing [Reagents](https://github.com/ocaml-multicore/reagents)

note: It was originally designed as kind of a low level library, providing a
multi-word compare-and-set primitive, for the purpose of implementing a
higher-level concurrent programming library called Reagents.

---

### Kcas now

- New obstruction-free
  [k-CAS-n-CMP algo](https://github.com/ocaml-multicore/kcas/blob/main/doc/gkmz-with-read-only-cmp-ops.md)
  - Includes [lock-free k-CAS](https://arxiv.org/abs/2008.02527) as subset
  - Extended with read-only CMPs
- Direct style transactional interface
  - Backed by splay tree
- [Nested transactions](https://github.com/ocaml-multicore/kcas#log-updates-optimistically)
  - Conditionals, Disjunctions
- [Blocking](https://github.com/ocaml-multicore/kcas#blocking-transactions) and
  [Timeouts](https://github.com/ocaml-multicore/kcas#timeouts)
- [A library of data structures](https://ocaml-multicore.github.io/kcas/doc/kcas_data/Kcas_data/index.html)
  - `Hashtbl`, `Queue`, `Stack`, `Dllist`, `Mvar`, ...

note: The recent work done on Kcas has extended the scope of the library
significantly, turning it into a proper software transactional memory
implementation, while also significantly improving the performance of the
library. Furthermore, Kcas now also comes with a library of data structures.

---

### Blog post

<p>
<a href="https://tarides.com/blog/2023-08-07-kcas-building-a-lock-free-stm-for-ocaml-1-2/"><img width="480" height="270" src="blog-1.webp"></a>
<a href="https://tarides.com/blog/2023-08-10-kcas-building-a-lock-free-stm-for-ocaml-2-2/"><img width="480" height="270" src="blog-2.webp"></a>
</p>

Kcas: Building a Lock-Free STM for OCaml

<a href="https://tarides.com/">tarides.com</a>

note: Before continuing with the introduction, I want to mention that there is a
two part blog post that discusses the recent work on Kcas comprehensively. You
might want to read that blog post to learn more about the library.

---

### Shared memory locations

```ocaml
let account_a = Loc.make 100
let account_b = Loc.make 200
```

```ocaml
# Loc.fetch_and_add account_a 50
- : int = 100
```

```ocaml
# Loc.get account_a
- : int = 150
```

<small>(`Loc` is like `Atomic` with some extra features.)</small>

note: Kcas provides an abstraction of shared memory locations or the `Loc`
module, which is essentially a superset of the OCaml Stdlib `Atomic` module,
which, in turn, provides an abstraction similar to the mutable `ref`erence cells
of ML.

---

### Direct style transactions

```ocaml
let transfer ~xt from_account to_account amount =
  if amount < 0 then
    invalid_arg "transfer: amount < 0";

  if amount <= Xt.get ~xt from_account then begin
    ignore (Xt.fetch_and_add ~xt from_account (-amount));
    ignore (Xt.fetch_and_add ~xt   to_account   amount );
  end
```

```ocaml
Xt.commit { tx = transfer account_a account_b 50 }
```

```ocaml
# Loc.get account_a, Loc.get account_b
- : int * int = (100, 250)
```

<small>(Labeled argument <code>~xt</code> stands for explicit transaction
log.)</small>

note: Kcas also provides direct style transactions over shared memory locations.
The `transfer` function on this slide is a transaction that can be used to move
an amount from one account to another atomically. The labelled `~xt` parameter
refers to the transaction log, which is explicitly passed by the transaction
function to all the operations that manipulate shared locations. To perform the
transaction, one calls the `commit` function, which requires the transaction
function to be polymorphic with respect to the transaction log, because the
transaction function may be called multiple times and the log must not be
leaked.

---

### Example: LRU Cache

```ml
type ('k, 'v) cache

val cache : int -> ('k, 'v) cache

val get_opt : ('k, 'v) cache -> 'k -> 'v option

val set_blocking : ('k, 'v) cache -> 'k -> 'v -> unit
```

<small>(Essentially a bounded hash table with Least-Recently-Used replacement
policy.)</small>

note: Let's look at a bit more realistic example of a least-recently-used cache,
which is essentially a kind of bounded hash table.

---

### LRU Cache (constructors)

```ocaml
type ('k, 'v) cache = {
  space: int Loc.t;
  table: ('k, 'k Dllist.node * 'v) Hashtbl.t;
  order: 'k Dllist.t;
}
```

```ocaml
let cache capacity = {
    space = Loc.make capacity;
    table = Hashtbl.create ();
    order = Dllist.create ();
  }
```

<small>(Composition of a hash table and doubly-linked list. <code>Dllist</code>
tracks access order.)</small>

note: A simple way to implement such a cache is to use a hash table and a
doubly-linked list. The doubly-linked list is used to track the order of
accesses and decide which bindings to drop in case the cache becomes full.

---

### LRU Cache (sequential)

```ocaml
let get_opt c key =
  Hashtbl.find_opt c.table key
  |> Option.map @@ fun (node, datum) ->
     Dllist.move_l node c.order; datum
```

```ocaml
let set_blocking c key datum =
  let node =
    match Hashtbl.find_opt c.table key with
    | None ->
      if 0 = Loc.update c.space (fun n -> max 0 (n-1))
      then Dllist.take_blocking_r c.order
           |> Hashtbl.remove c.table;
      Dllist.add_l key c.order
    | Some (node, _) ->
      Dllist.move_l node c.order; node in
  Hashtbl.replace c.table key (node, datum)
```

<small>(Individual ops are concurrency and parallelism safe, but composition is
not.)</small>

note: What is important here is that both the hash table and the doubly-linked
list are manipulated by the operations, which is so common in sequential
programming that one doesn't even think about it. The problem is that this is
not concurrency-safe even though the individual operations on hash tables and
doubly-linked lists happen to be concurrency-safe, because they come from the
data structure library of Kcas. So, how do we make this safe?

---

### LRU Cache (sequential)

```ocaml
let get_opt     c key =
  Hashtbl.   find_opt     c.table key
  |> Option.map @@ fun (node, datum) ->
     Dllist.   move_l     node c.order; datum
```

```ocaml
let set_blocking     c key datum =
  let node =
    match Hashtbl.   find_opt     c.table key with
    | None ->
      if 0 = Loc.update    c.space (fun n -> max 0 (n-1))
      then Dllist.   take_blocking_r     c.order
           |> Hashtbl.   remove     c.table;
      Dllist.   add_l     key c.order
    | Some (node, _) ->
      Dllist.   move_l     node c.order; node in
  Hashtbl.   replace     c.table key (node, datum)
```

<small>(With a few extra spaces to align things.)</small>

note: Well, obviously the first step is to add a few extra spaces.

---

### LRU Cache (transactional)

```ocaml
let get_opt ~xt c key =
  Hashtbl.Xt.find_opt ~xt c.table key
  |> Option.map @@ fun (node, datum) ->
     Dllist.Xt.move_l ~xt node c.order; datum
```

```ocaml
let set_blocking ~xt c key datum =
  let node =
    match Hashtbl.Xt.find_opt ~xt c.table key with
    | None ->
      if 0 = Xt.update ~xt c.space (fun n -> max 0 (n-1))
      then Dllist.Xt.take_blocking_r ~xt c.order
           |> Hashtbl.Xt.remove ~xt c.table;
      Dllist.Xt.add_l ~xt key c.order
    | Some (node, _) ->
      Dllist.Xt.move_l ~xt node c.order; node in
  Hashtbl.Xt.replace ~xt c.table key (node, datum)
```

<small>(Safe transactional version.)</small>

note: And then we fill the blanks to turn the functions into transactions by
passing the transaction log to transactional versions of the operations on the
hash table and doubly-linked list. As cute as this may appear, this is not too
far from the truth that transactions basically allow you to continue using
familiar sequential looking ways to structure concurrent code. Before we move to
the next slide, note that the `get_opt` transaction is non-blocking, and returns
an option, and the `set_blocking` transaction is blocking, such that it will not
return in case the capacity of the cache is zero.

---

### LRU Cache (further)

```ocaml
let get_blocking ~xt c key =
  match get_opt ~xt c key with
  | None -> raise_notrace Retry.Later
  | Some datum -> datum
```

```ocaml
let try_set ~xt c key datum =
  match set_blocking ~xt c key datum with
  | () -> true
  | exception Retry.Later -> false
```

```ocaml
let get_if ~xt c key predicate =
  let snap = Xt.snapshot ~xt in
  let datum = get_blocking ~xt c key in
  if predicate datum then datum
  else Retry.later (Xt.rollback ~xt snap)
```

<small>(These do not depend on LRU cache internals.)</small>

note: What we see here is that we can convert between blocking and non-blocking
transactions. First we convert the non-blocking `get`. In case we couldn't
return some value we simply raise the `Retry.Later` exception, which signals to
the `commit` mechanism that the transaction should be retried only after some
locations accessed by the transaction have changed. Dually, we convert the
blocking `set` to a non-blocking transaction by handling the `Retry.Later`
exception. Finally, we build upon the blocking version of `get` to create a
conditional `get` that blocks until the value passes a given predicate. We do
this by explicitly scoping changes by taking a snapshot of the transaction log
and rolling it back before we signal retry.

---

### LRU Cache (example)

```ocaml
let cache_a = cache 7
let cache_b = cache 6
```

```ocaml
let domain = Domain.spawn @@ fun () ->
  let tx = Xt.first [
    get_if cache_a "x" (fun x -> x <= 0);
    get_if cache_b "y" (fun y -> y  > 0); ] in
  Xt.commit { tx }
```

```ocaml
Xt.commit { tx = set_blocking cache_b "y" 0 };
Xt.commit { tx = set_blocking cache_b "y" 76 };
```

```ocaml
# Domain.join domain
- : int = 76
```

note: Putting it all together, here is a simple example that creates a couple of
caches and spawns a domain that blocks to conditionally get an element from one
of the caches. Once a cache has been populated with a passing value, the domain
completes.

---

~Introduction~

▸ **_Anatomy of a transaction_**

Scheduler agnosticism

Atomicity and Isolation

Performance

Conclusion

note: Next we are going to look at how transactions work under the hood.

---

### Shared memory location (1/5)

<img width="60%" src="loc-u.svg">

<small>(The value of <code>x</code> is undetermined, but reader can help to
determine it via the log.)</small>

note: But first we take a look at how locations are represented. As usual, the
trick is a level of indirection. This diagram shows a location named `x`, whose
value is undetermined. The state of a location supports this possibility by
storing two values, the values `0` and `2` in this case, and by pointing to a
transaction marked as `Undetermined`.

---

### Shared memory location (2/5)

<img width="60%" src="loc-a-i.svg">

<small>(The value of <code>x</code> is determined to <code>After</code>, but
leaks some space.)</small>

note: What we see here is the same location `x`, but this time the value of the
location has been determined to be the `After` value. Note that the state still
includes the `Before` value. If the values were pointers, this could be a
possible space leak.

---

### Shared memory location (3/5)

<img width="60%" src="loc-a.svg">

<small>(The value of <code>x</code> is determined to
<code>After</code>.)</small>

note: In this diagram we see that both the `Before` and `After` values are the
same and the transaction state has also been inlined to the location state.
While this representation is slightly redundant, there is no leak.

---

### Shared memory location (4/5)

<img width="60%" src="loc-b-i.svg">

<small>(The value of <code>x</code> is determined to <code>Before</code>, but
leaks some space.)</small>

note: It could also happen that the value of the location is determined to be
the `Before` value.

---

### Shared memory location (5/5)

<img width="60%" src="loc-b.svg">

<small>(The value of <code>x</code> is determined to
<code>Before</code>.)</small>

note: And then the extra space could also be released similarly.

---

### A transaction

```ocaml
let x = Loc.make 1
let y = Loc.make 3
let d = Loc.make 2
```

```ocaml
let tx ~xt =
  let d' = Xt.get ~xt d in
  ignore (Xt.fetch_and_add ~xt x   d' );
  ignore (Xt.fetch_and_add ~xt y (-d'));
```

```ocaml
Xt.commit { tx }
```

```ocaml
# Loc.get x, Loc.get y, Loc.get d
- : int * int * int = (3, 1, 2)
```

note: Now here is a simple transaction that reads the location `d` and then adds
the value of `d` to the location `x` and subtracts it from the location `y`.
Let's look at how this transaction runs in more detail.

---

### Phases

1. **_Construct a log_** of CMPs and CASes
2. **_Execute_** CMPs and CASes
3. (Optionally) **_Verify_** CMPs
4. Final CAS to **_determine result_**
5. **_Release space_** to avoid leaks

note: As an overview, here is list of phases that a transaction goes through. We
will now go through the phases.

---

#### 1. `Initial`

<img width="80%" src="transaction-0.svg">

<small>(Top is transaction log. Bottom is shared memory. Red is write.)</small>

note: This is the initial state at the beginning of our transaction. We have the
shared memory locations `d`, `y`, and `x` at the bottom in the order of their
ids. At the top we have the transaction log, which is marked as `Undetermined`
and is currently empty. An empty splay tree to be more precise.

---

#### 1. `let d' = Xt.get ~xt d`

<img width="80%" src="transaction-1-i.svg">

<small>(The <code>d</code> location is read to determine
<code>d=2</code>.)</small>

note: Then we start to build the transaction log. The first operation we need to
record is the read of the `d` location. We see words of the location `d` being
read marked in blue.

---

#### 1. `let d' = Xt.get ~xt d`

<img width="80%" src="transaction-1.svg">

<small>(An entry is added to the log corresponding to the read of
<code>d</code>.)</small>

note: Then we create an entry, a splay tree node, in the transaction log
corresponding to the read in the form of a compare operation. The red words are
being written to.

---

#### 1. `Xt.fetch_and_add ~xt x d'`

<img width="80%" src="transaction-2-i.svg">

<small>(The <code>x</code> location is read to determine
<code>x=1</code>.)</small>

note: Then we read the location `x` for the fetch-and-add operation.

---

#### 1. `Xt.fetch_and_add ~xt x d'`

<img width="80%" src="transaction-2.svg">

<small>(An entry is added to the log corresponding to the write
<code>x:=3</code>.)</small>

note: And add an entry to the transaction log corresponding to the
fetch-and-add. Notice that we now created a compare-and-set node. We can tell
that this is the case, because we allocated a new state record with the values
`1` and `3` and with a pointer back to the root of the transaction log. The
previous compare operation on the `d` location simply points to the state of the
`d` location, which, crucially, does not point to our transaction log.

---

#### 1. `Xt.fetch_and_add ~xt y (-d')`

<img width="80%" src="transaction-3-i.svg">

<small>(The <code>y</code> location is read to determine
<code>y=3</code>.)</small>

note: Then we do the read of the `y` location.

---

#### 1. `Xt.fetch_and_add ~xt y (-d')`

<img width="80%" src="transaction-3.svg">

<small>(An entry is added to the log corresponding to the write
<code>y:=1</code>. The log is complete.)</small>

note: And add the compare-and-set operation to the transaction log like we did
with the `x` location. The transaction log is now complete and we move to the
execution phase.

---

#### 2. `CMP d`

<img width="80%" src="transaction-4.svg">

<small>(The log entry corresponding to read of <code>d</code> is checked by
identity &mdash; avoids ABA problem.)</small>

note: In the execution phase we go through the log in the order of the ids of
the locations, which we can do by traversing the splay tree in order. The first
operation is the compare of `d`. To execute the compare operation, we will just
verify that the location `d` still points to the same state that we captured in
the log. Note that we compare the state by reference, which conveniently avoids
ABA-problems.

---

#### 2. `CAS y`

<img width="80%" src="transaction-5-i.svg">

<small>(The log entry corresponding to the write of <code>y</code> is checked by
value.)</small>

note: To execute the compare-and-set operation on `y`, we first check that the
location `y` still has the same value as `Before`. And it does, so will execute
a compare-and-set to change `y` to point to the new state in the log.

---

#### 2. `CAS y`

<img width="80%" src="transaction-5.svg">

<small>(The location <code>y</code> is updated to point to new state. Now the
log is public.)</small>

note: And here we see the `y` location updated. Note that we did not update the
old state of `y`, we simply updated the location to point to the new state.
Also, the value of location `y` is now `Undetermined` and it points to our
transaction log. This means that any concurrent access of `y` will use the
transaction log to execute the transaction to determine the value of `y`. Just
like we are doing.

---

#### 2. `CAS x`

<img width="80%" src="transaction-6-i.svg">

<small>(The log entry corresponding to the write of <code>x</code> is
checked.)</small>

note: The last operation to execute is the compare-and-set of `x`. First we
compare.

---

#### 2. `CAS x`

<img width="80%" src="transaction-6.svg">

<small>(The location <code>x</code> is updated to point to new state.)</small>

note: And then we perform the compare-and-set and update the `x` location. We
have now executed all the operations recorded in the transaction log.

---

#### 3. `Verify`

<img width="80%" src="transaction-7.svg">

<small>(Before marking the log as determined, reads must be verified.)</small>

note: The next phase is to verify the compare operations. So, we go through the
log again, identify the compare operations, which do not point back to our
transaction log, and verify that the compared locations still have their
original states. Once the verify phase is complete, we can perform the final
compare-and-set and determine the values of all the currently `Undetermined`
locations.

---

#### 4. `Final CAS (success)`

<img width="80%" src="transaction-8.svg">

<small>(The log is marked as determined successfully. Log is no longer
public.)</small>

note: At this point the values of the locations we modified are marked as
determined, but they now contain values that are no longer needed.

---

#### 5. `Release (success)`

<img width="80%" src="transaction-9.svg">

<small>(Finally any extra memory is released.)</small>

note: So, as the last step, we go through the transaction log once more,
overwrite the unused values and "inline" the transaction state to the locations
we modified. What we saw here was the successful or happy path case. In case of
failure, however...

---

#### 4. `Final CAS (failure)`

<img width="80%" src="transaction-10.svg">

<small>(In case of failure, log is marked as determined unsuccessfully.)</small>

note: we simply mark the transaction as determined unsuccesfully and all the
locations that we managed to modify, whether we managed to modify any, some, or
all of them, will then have the values they had `Before`.

---

#### 5. `Release (failure)`

<img width="80%" src="transaction-11.svg">

<small>(And any extra memory is released.)</small>

note: And then we will also need to perform a release step symmetrically to the
successful case.

---

### Notes

- `Loc`s are not mutated during log construction
- Splay tree construction often linear time
- Verify only needed when CMP is followed by CAS

- Details skipped

  - Handling of awaiters

    - Copied just before CAS operations
    - Called during release step

  - Periodic transaction log verification

  - ...

note: The previous diagrams and discussion omitted some minor details. In
particular, we didn't discuss blocking and the handling of awaiters. We also
ignored details related to the retry mechanism.

---

~Introduction~

~Anatomy of a transaction~

▸ **_Scheduler agnosticism_**

Atomicity and Isolation

Performance

Conclusion

note: But let's move to the next topic.

---

### Kcas is scheduler agnostic

Non-blocking algorithms

- Require very little from scheduler
  - Obstruction-free
    - progress if given time in isolation
  - Lock-free
    - one thread will make progress
- Work also in contexts where locks do not

_But what about Blocking and Timeouts?_

note: As Kcas employs non-blocking algorithms, it requires very little from the
scheduler and can also work in contexts, such as signal handlers, where locks
would be inappropriate. However, some features of Kcas, namely blocking and
timeouts, require help from the scheduler.

---

### One does not simply

- Suspend and resume

- Execute an action after given time

_Because "How" depends on context:_

- "Plain" Domain or (Sys)Thread?
  - These have no handlers!
- Fiber?
  - Scheduler _specific_ effect or function!

note: The problem is that one cannot simply suspend and resume a fiber or
request that an action would be executed after a given period of time, because
the way such concurrent runtime services need to be performed depends on the
context. Perhaps in the future there are standard effects for some of these,
but, today, we don't, and we need a solution that works not only with specific
effects based schedulers, but also with plain domains and systhreads that do not
have any handlers.

---

### A level of indirection?

- Idea: Use domain (or thread) local storage

  - Use same dynamic binding structure as with handlers

- Default mechanisms can be provided in the absence of a scheduler

- Effects based schedulers can install their own implementations

- The mechanism for the current context can then invoked dynamically

note: So, how do we solve this problem? Perhaps a level of indirection does the
trick. What if we store such mechanisms in domain or thread local variables with
essentially the same kind of dynamic binding structure as effect handlers? This
allows default mechanisms not based on effects and also allows schedulers to
install their own implementations of the mechanisms, without having to agree on
exact effect type constructors. and libraries like Kcas can then simply
dynamically invoke the mechanisms installed for the current dynamic context.

---

### Interfaces

```ml
Domain_local_await : sig
  type t = { release : unit -> unit; await : unit -> unit }
  val prepare_for_await : unit -> t

  val using : prepare_for_await:(unit -> t) ->
              while_running:(unit -> 'a) -> 'a
end
```

```ml
Domain_local_timeout : sig
  val set_timeoutf : float -> (unit -> unit) -> unit -> unit

  val using :
    set_timeoutf:(float -> (unit -> unit) -> unit -> unit) ->
    while_running:(unit -> 'a) -> 'a
end
```

<small>(The <code>using</code> functions are used by schedulers to install their
own implementations.)</small>

note: This is what I ended up with. The domain-local-await service allows to
suspend and resume the current thread of execution and the domain-local-timeout
service allows to request an action to be performed after specified time in
seconds. The `using` functions are for schedulers to install their own
implementations &mdash; much like one would install an effect handler. To use
the services one simply calls the `prepare_for_await` or the `set_timeoutf`
function.

---

### A scheduler friendly `sleepf`

```ocaml
let sleepf seconds =
  let Domain_local_await.{ release; await } =
    Domain_local_await.prepare_for_await ()
  in
  let cancel =
    Domain_local_timeout.set_timeoutf seconds release
  in
  try await ()
  with cancellation_exn ->
    cancel ();
    raise cancellation_exn
```

note: Here is an example of a scheduler friendly sleep function. It first uses
the domain-local-await service to obtain a pair of actions. One action to
`release` and another to `await`. The domain-local-timeout service is then used
to request the `release` action to be called after the desired number of
seconds. Then the `await` action is called, which suspend the current thread of
execution until either the `release` action is called or the current thread of
execution is cancelled, in which case `await` raises an exception and we
`cancel` the timeout to avoid leaks. You might wonder why should one bother with
something like this. Currently the OCaml Stdlib `Unix.sleepf` function is
essentially unusable with effects based schedulers, because it blocks the entire
domain or thread such that no other fiber can be scheduled to run on the domain
or thread. This has already caused surprises to people interested in trying out
multicore OCaml.

---

### Concurrent runtime services?

- DLA and DLT are examples of such

- Others: Fibers, Nested parallelism, Cancellation, ...

- Many libraries (Kcas, Saturn, ...) can be scheduler independent

  - And can even work across scheduler in an app

    - Example on
      [discuss](https://discuss.ocaml.org/t/interaction-between-eio-and-domainslib-unhandled-exceptions/11971/10)

  - DLA + Threads can e.g. implement async IO
    - Examples of lazy, mutex, and IO in
      [DLA repository](https://github.com/ocaml-multicore/domain-local-await/#contents)

- Applications can pick scheduler(s) and libraries

<small>(kcas-raw, kcas-eio, kcas-miou, kcas-moonpool,
<b>kcas-one-ring-to-rule-them-all</b>?)</small>

note: More generally, I'd like to propose that concurrent schedulers and
libraries in multicore OCaml would agree on a set of concurrent runtime services
for interoperability. This would allow many libraries requiring those services
to be written in a scheduler agnostic manner. This would even make it possible
to implement transparently asynchronous IO in a scheduler agnostic manner. In
fact, I have implemented a proof-of-concept of this and several other currently
lacking features of multicore OCaml as examples in the domain-local-await
library documentation. Now imagine a world where we didn't have the
interoperability. Libraries like Kcas would need to parameterized, in some
cumbersome manner, or would need to be specialized to each and every scheduler.

---

<img width="70%" src="Sauron-Ring-Lord-of-the-Rings.jpg">

Choose **_interoperability_** for the greater good!

note: I wish I had more time to elaborate on this, but I'd really love to see
OCaml having an ecosystem of interoperable rather than community dividing
concurrent libraries. Currently Eio, Domainslib, and Moonpool support
domain-local-await and Kcas should work on all of those. In the near future, I
expect Saturn, our lock-free data structure library, to also start using
domain-local-await.

---

~Introduction~

~Anatomy of a transaction~

~Scheduler agnosticism~

▸ **_Atomicity and Isolation_**

Performance

Conclusion

note: Let's then move on to discuss a trade-off that Kcas makes.

---

### Kcas guarantees

> Transaction can only see succesfully committed values and may only commit
> successfully after it has seen a consistent snapshot of shared memory
> locations.

But...

note: The algorithms underlying Kcas have many nice properties. In particular,
updates to memory are disjoint access parallel and there is no global sequential
bottleneck during updates. Most importantly, a transaction may only commit
successfully in case it observed a consistent snapshot of memory. However...

---

### Kcas guarantees not

> It is possible for a transaction to see an inconsistent view with values
> committed by multiple different transactions.

What does that mean?

note: it is possible for a transaction to see an inconsistent view of shared
memory locations with values committed by multiple different transactions.
Having seen such an inconsistent view, a transaction cannot commit successfully
and must be retried. Unfortunately that can still cause problems.

---

### Invariants may appear broken

```ocaml
let unsafe_subscript ~xt xs i =
  let xs' = Xt.get ~xt xs
  and i' = Xt.get ~xt i in
  xs'.(i')
```

`xs'` and `i'` may come from different transactions!

note: Here is an example of a problematic transaction. The idea is that the two
locations `xs` and `i` are always updated together and that accessing the `xs`
array with the index `i` is always safe. Unfortunately, it is possible for a
particular attempt of the transaction function to get the values of `xs` and `i`
from two different transactions. This means that the array access could be out
of bounds.

---

### Unique?

- Several STMs have same property, e.g.

  - Haskell,
  - Scala Zio, and
  - [Wyatt-STM](https://github.com/bretthall/Wyatt-STM) (C++).

note: This property of Kcas isn't unique. While there are algorithms for
software transactional memory that do not allow transactions to observe such
inconsistencies, several software transactional memory implementations have
chosen to use algorithms that do allow it. One of the reasons for that is that
the known algorithms to rule out such inconsistencies tend to be relatively
expensive.

---

### Mitigation

Kcas validates the log periodically on location accesses.

Effective in transactions where any loops access locations and there are no
partial ops.

> IMSE, one very rarely needs loops or unsafe operations in "user" transactions.
> IOW, when just composing things using carefully implemented data structures,
> transactions just work.

But this is ineffective against partial operations depending on invariants.

And in such cases one needs to validate accesses explicitly.

note: To mitigate the problem, Kcas implements an efficient periodic validation
scheme. This means that transactions that only record updates of shared memory
locations and do not perform partial operations relying on invariants between
shared memory locations are safe. Unfortunately, periodic validation is not
effective against problems caused by partial operations relying on invariants
between multiple locations. For those cases...

---

### Explicit validation

```ocaml
let subscript ~xt xs i =
  let xs' = Xt.get ~xt xs
  and i' = Xt.get ~xt i in
  (* Validate accesses after making them: *)
  Xt.validate ~xt xs;
  Xt.validate ~xt i;
  xs'.(i')
```

<small>(Invariant between <code>xs'</code> and <code>i'</code> holds after
<code>validate</code> calls.)</small>

note: Kcas provides an explicit validation mechanism that allows one to validate
that locations recorded in the transaction log have not been updated outside of
the transaction, which ensures that any invariants between such locations will
hold.

---

~Introduction~

~Anatomy of a transaction~

~Scheduler agnosticism~

~Atomicity and Isolation~

▸ **_Performance_**

Conclusion

note: Before concluding, let's briefly discuss another elephant in the room
&mdash; performance.

---

### Optimizations

Kcas and the data structures using it have been fairly carefully optimized.

- Using bit manipulation tricks
- Making careful representation choices
- Avoiding indirections
- Avoiding contention
- Avoiding false-sharing
- Avoiding unnecessary fences
- ...

But there are more indirections and allocations than with plain Atomics.

note: While I have spent a great deal of effort optimizing Kcas, it is clear
that transactions have some unavoidable overheads in the form of additional
indirections, allocations, and memory accesses compared to the use of plain
atomics. How does Kcas actually perform?

---

### Queues

| config |        Kcas |      Saturn | Stdlib |
| -----: | ----------: | ----------: | -----: |
|  1 + 1 |        27.0 | <u>33.5</u> |    9.7 |
|  1 + 2 |        24.5 | <u>27.4</u> |    3.2 |
|  2 + 1 | <u>31.6</u> |        24.5 |    4.9 |
|  2 + 2 | <u>24.6</u> |        22.1 |    9.7 |

Adders + Takers, M msgs/s

note: Here are results from a set of queue benchmarks that have a single shared
queue accessed by multiple domains running in parallel. The Stdlib version
naïvely uses a Mutex to protect a sequential queue. The Saturn version uses a
lock-free Michael-Scott queue implementation. As can be seen, the Kcas Queue
does not perform all that badly. In this benchmark, Kcas even beats the
lock-free Michael-Scott queue of the Saturn library in a couple of
configurations. I should note, however, that I have also written a much faster
implementation of the lock-free Michael-Scott queue in OCaml. With that said,
the performance of the Kcas queue should be unlikely to be the bottleneck in
most practical situations.

---

### Stacks

| config | Kcas |      Saturn |
| -----: | ---: | ----------: |
|  1 + 1 | 29.5 | <u>42.7</u> |
|  1 + 2 | 28.3 | <u>38.7</u> |
|  2 + 1 | 31.7 | <u>44.8</u> |
|  2 + 2 | 29.0 | <u>39.9</u> |

Pushers + Poppers, M msgs/s

note: Here is a similar benchmark using a Treiber stack. With such a simple data
structure the higher overhead of Kcas does show. But I'd still argue that the
performance of Kcas should be fine for most use cases.

---

### Hash tables

|  config |         Kcas |      Stdlib |
| ------: | -----------: | ----------: |
| 1 \* 90 |         35.4 | <u>38.3</u> |
| 2 \* 90 |  <u>62.0</u> |        16.9 |
| 4 \* 90 | <u>108.7</u> |        14.5 |
| 1 \* 10 |         10.4 | <u>37.0</u> |
| 2 \* 10 |  <u>19.0</u> |        14.9 |
| 4 \* 10 |  <u>31.1</u> |        11.7 |

Domains \* percentage reads, M ops / s

note: As the last benchmark, here are results from a hash table benchmark. Like
with the earlier queue benchmark, the Stdlib version naïvely uses a mutex to
protect the hash table. What can be seen here is that while the Kcas hash table
has higher overhead for writes, it can outperform naïve locking already at just
two parallel domains. Reads with the Kcas hash table can proceed in parallel
without interference and scale well.

---

### Essentially

These are apples to oranges &mdash; Kcas provides more features.

Kcas should give reasonably good performance.

Overhead with Kcas is higher &mdash; even without txs.

Overhead evens out with contention.

Easier to use better algorithms.

Lots of work done to make performance stable and scalable.

note: Always take benchmarks with a cup of salt. There is still a lot of work to
do to benchmark Kcas and other approaches. While it is clear that Kcas has some
overheads, it also seems to be able to perform reasonably well. Furthermore,
Kcas and the associated data structure library has been engineered to avoid
common performance pitfalls such as false sharing. While the implementation of
high performance data structures takes effort and insight even with Kcas, it is
typically much easier than the implementation of similar data structures using
atomics. And, with Kcas, you get composability, blocking, and timeouts.

---

~Introduction~

~Anatomy of a transaction~

~Scheduler agnosticism~

~Atomicity and Isolation~

~Performance~

▸ **_Conclusion_**

---

### Summary

- Kcas is a STM for OCaml.
- Based on a new k-CAS-n-CMP algorithm.
- Provides a familiar programming model.
- Comes with a library of data structures.
- Allows expressive blocking with timeouts.
- Gives scalable performance.
- Is scheduler agnostic.
- Has examples and documentation.
- 1.0 version coming soon.

note: In summary, Kcas is a software transactional memory implementation for
OCaml. It is based on a new and efficient k-CAS-n-CMP algorithm wrapped behind a
convenient direct style interface. It comes with a library of composable,
reasonably well performing, concurrent data structures. And it supports
scheduler friendly blocking and timeouts.

---

Concurrency with transactions is fun!

Choose interoperability!

---

<img width="30%" src="kcas.svg">

Kcas

note: Thank you! Questions?

---

### STM Trade-offs

- Kcas
  - lock-free and disjoint access parallel
  - strictly serializable
  - transactions may observe read-skew
- With [opacity](https://doi.org/10.1145/1345206.1345233) (e.g. TL 2)
  - least surprises
  - often lock-based, dependent on scheduler
  - global bottleneck (version clock) potentially limits scalability
- Snapshot isolation
  - write skew
  - difficult to reason about
