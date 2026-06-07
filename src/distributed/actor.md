# Actor Model

The Actor model (Hewitt, 1973) is a concurrency paradigm where "everything is an actor." An actor is a computational entity that:
- Has a mailbox (queue of messages).
- Processes one message at a time, sequentially.
- Can send messages to other actors.
- Can create new actors.
- Can change its behavior for the next message.

Actors communicate only through message passing — no shared memory, no locks, no atomics. This eliminates race conditions within an actor (single-threaded processing) and makes the model naturally distributed: two actors can be on different machines, and message passing works the same way.

## Erlang and the BEAM VM

Erlang (Ericsson, 1986) is the canonical actor language. Originally built for telephone switches (five 9s of reliability: 99.999% uptime ≈ 5 minutes downtime/year), Erlang's actor model has proven itself in production over decades.

```erlang
% Counter actor
-module(counter).
-export([start/0, increment/1, get/1, stop/1]).

start() ->
    spawn(fun() -> loop(0) end).  % Create actor with initial state 0

loop(Count) ->
    receive
        {increment, From} ->
            From ! {ok, Count + 1},       % Respond to sender
            loop(Count + 1);
        {get, From} ->
            From ! {value, Count},
            loop(Count);
        stop ->
            ok                              % Terminate
    end.
```

Usage:

```erlang
Counter = counter:start(),
Counter ! {increment, self()},  % Send increment message
receive {ok, NewCount} -> io:format("Count: ~p~n", [NewCount]) end.
```

Each actor is a process (Erlang term, not OS process). An Erlang process uses ~300 bytes of memory initially and is scheduled by the BEAM VM's preemptive scheduler (each process gets ~2000 reductions before being preempted). A single BEAM VM can run millions of processes. This is the key to Erlang's reliability: isolate every component as an actor, and if one crashes, the rest continue.

## Supervision and Fault Tolerance

The "let it crash" philosophy: don't write defensive code for every possible error. Let the actor crash, and have a **supervisor** restart it in a clean state.

```erlang
-module(my_supervisor).
-behaviour(supervisor).

init(_Args) ->
    SupFlags = #{strategy => one_for_one, intensity => 5, period => 60},
    ChildSpecs = [
        #{id => counter, start => {counter, start, []}, restart => permanent}
    ],
    {ok, {SupFlags, ChildSpecs}}.
```

If the counter actor crashes, the supervisor restarts it. If it crashes 5 times in 60 seconds, the supervisor itself crashes (escalation). This creates a supervision tree: workers at the leaves, supervisors at internal nodes, and a root supervisor at the top. Every component has a supervisor responsible for its lifecycle.

This model achieves nine 9s of reliability (99.9999999% uptime, < 32 ms downtime/year) in Ericsson's AXD301 ATM switch — 1.7 million lines of Erlang, 99.9999999% reliability over years of operation.

## Akka (Scala/Java)

Akka brings the Actor model to the JVM. Actors are objects with mailboxes:

```scala
import akka.actor._

class CounterActor extends Actor {
    var count = 0
    
    def receive = {
        case "increment" =>
            count += 1
        case "get" =>
            sender() ! count
    }
}

val system = ActorSystem("MySystem")
val counter = system.actorOf(Props[CounterActor], "counter")

counter ! "increment"
counter ! "increment"

// Ask pattern: send message, get Future response
implicit val timeout: Timeout = 3.seconds
val future: Future[Int] = (counter ? "get").mapTo[Int]
// future = 2
```

Akka adds:
- **Location transparency**: actors can be local or remote. The code for sending a message is identical.
- **Clustering**: actors on different machines form a cluster. Messages are serialized and sent over the network.
- **Persistence**: actor state can be persisted (event sourcing). On restart, the actor replays events to reconstruct state.
- **Streams**: Akka Streams for reactive stream processing with backpressure.

## Message Ordering

Actors guarantee message delivery (no lost messages) but not message ordering across different actors. If actor A sends m1 then m2 to actor B, B receives them in order. But if A sends m1 to B and C, and C sends m2 to B upon receiving m1, B may receive m2 before m1 (if C processes m1 and sends m2 before m1 reaches B).

This is the same causal ordering problem from distributed systems. Erlang's solution: don't rely on ordering across actors. Design protocols where message ordering within an actor pair is sufficient.

## When to Use Actors

**Best for:**
- Systems with many independent stateful components (chat servers, multiplayer games, IoT device management).
- Systems requiring fault tolerance (telecom, financial systems, industrial control).
- Systems where the Actor model naturally matches the domain (each IoT device = an actor, each chat room = an actor).

**Not ideal for:**
- CPU-bound parallelism (actors add message-passing overhead for data that could be shared). Use threads/OpenMP/CUDA.
- Purely functional transformations (MapReduce/Spark are simpler).
- Low-latency request-response where the actor mailbox adds ~100 ns overhead (use direct function calls or lock-free queues).

## Key Lessons

1. **Actors eliminate shared-state concurrency bugs.** No mutexes, no atomics, no memory ordering. All communication is via messages. The price: message-passing overhead and the need to design protocols.
2. **"Let it crash" is a design philosophy, not laziness.** Supervision trees provide fault tolerance without defensive code. The supervisor handles recovery; the actor handles the happy path.
3. **Erlang's per-actor overhead is negligible (~300 bytes).** This enables millions of actors — one per connection, one per device, one per session. This is impossible with OS threads.
4. **The Actor model is naturally distributed.** Since actors communicate only via messages, splitting them across machines doesn't change the programming model. Akka Cluster and Erlang's distributed Erlang make this transparent.
5. **Message ordering guarantees are limited.** Within an actor pair, messages are ordered. Across actors, causal ordering requires explicit protocols (Lamport timestamps, vector clocks).
