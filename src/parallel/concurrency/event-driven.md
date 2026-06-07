# Event-Driven Concurrency

Event-driven concurrency is the oldest concurrency model still in wide use. Instead of creating a thread per task, a single thread loops through pending events and dispatches them to handlers. This is how JavaScript (in the browser and Node.js), Nginx, Redis, and most GUI frameworks work.

## The Event Loop

```c
while (running) {
    event = get_next_event();  // From epoll, kqueue, or a queue
    handler = find_handler(event.type);
    handler(event);
}
```

The event loop is single-threaded. Each handler runs to completion without being preempted. This eliminates race conditions — no two handlers run concurrently. The tradeoff: a handler that takes too long blocks the entire loop. A 100ms computation in a handler freezes all other clients for 100ms.

## JavaScript's Event Loop

JavaScript in the browser and Node.js is the most visible event-driven system:

```javascript
// Browser: user clicks a button
document.getElementById('my-button').addEventListener('click', () => {
    console.log('Button clicked!');
    // This runs to completion. No other JS runs until it finishes.
});

// Node.js: handle an HTTP request
const server = http.createServer((req, res) => {
    // This callback handles one request
    fs.readFile('large_file.txt', (err, data) => {
        // This runs AFTER the file is read (I/O callback)
        res.end(data);
    });
    // The server can handle other requests while waiting for the file
});
server.listen(8080);
```

The key insight: I/O is asynchronous. `fs.readFile` doesn't block the event loop — it registers a callback and returns immediately. When the file is ready, the callback is queued and executed on the next event loop iteration. This allows one thread to handle thousands of concurrent connections.

The event loop processes events in order:
1. Execute synchronous code (the current "tick").
2. Process microtasks (Promise callbacks, `queueMicrotask`).
3. Process macrotasks (setTimeout, setInterval, I/O callbacks).
4. Repeat.

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);  // Macrotask
Promise.resolve().then(() => console.log('3'));  // Microtask
console.log('4');
// Output: 1, 4, 3, 2
```

Microtasks run before the next macrotask, even if the macrotask was scheduled first. This is why Promises resolve before `setTimeout(0)`.

## Reactor Pattern (epoll, kqueue, IOCP)

On Linux, the event loop uses `epoll` to monitor thousands of file descriptors:

```c
int epoll_fd = epoll_create1(0);
struct epoll_event ev;
ev.events = EPOLLIN;  // Readable
ev.data.fd = socket_fd;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, socket_fd, &ev);

struct epoll_event events[MAX_EVENTS];
while (running) {
    int n = epoll_wait(epoll_fd, events, MAX_EVENTS, timeout_ms);
    for (int i = 0; i < n; i++) {
        if (events[i].events & EPOLLIN) {
            handle_read(events[i].data.fd);
        }
        if (events[i].events & EPOLLOUT) {
            handle_write(events[i].data.fd);
        }
    }
}
```

`epoll_wait` blocks until at least one fd is ready. It returns only the ready fds — no scanning of the entire set. This is O(1) in the number of fds (technically O(ready fds), not O(total fds)). Nginx uses this to handle 100,000+ concurrent connections with a single thread.

## The Actor Model (Erlang/Akka)

The Actor model takes event-driven concurrency and distributes it across multiple processes (Erlang) or threads (Akka). An actor is an entity that:
- Has a mailbox (queue of incoming messages).
- Processes one message at a time.
- Can send messages to other actors.
- Can create new actors.

```scala
// Akka (Scala)
class CounterActor extends Actor {
    var count = 0
    
    def receive = {
        case "increment" => count += 1
        case "get" => sender() ! count
    }
}

val counter = system.actorOf(Props[CounterActor], "counter")
counter ! "increment"
counter ! "increment"
val future = counter ? "get"  // Returns Future[Int] = 2
```

Each actor processes messages sequentially — no race conditions within an actor. But multiple actors run concurrently on different threads, so messages between actors can interleave arbitrarily. The Actor model's strength: it makes concurrency explicit through message passing rather than shared memory.

Erlang takes this further: actors (called "processes") are so lightweight (300 bytes each) that a single Erlang VM can run millions. If an actor crashes, it doesn't affect others — supervisors can restart failed actors. This is the foundation of Erlang's legendary reliability (nine 9s: 99.9999999% uptime).

## Performance Comparison (handling 100K concurrent HTTP connections)

| Model | Threads/Cores | Memory/Connection | Latency at P99 |
|-------|---------------|-------------------|----------------|
| Thread-per-connection | 100,000 | 8 MB × 100K = 800 GB | 5 ms |
| Thread pool (100 threads) | 100 | 8 MB × 100 + queue | 50 ms (queuing) |
| Event loop (Node.js) | 1 | ~10 KB × 100K = 1 GB | 2 ms |
| Actors (Erlang) | 8 | ~300 B × 100K = 30 MB | 1 ms |
| Goroutines (Go) | 8 | ~2 KB × 100K = 200 MB | 1 ms |

Event-driven and actor models dominate for high-concurrency I/O. The thread-per-connection model is dead for this use case — it was never viable for more than ~1000 connections.

## Key Lessons

1. **Single-threaded event loops eliminate race conditions.** No two handlers run concurrently, so shared state is safe. The cost: long-running handlers block the loop.
2. **I/O is always asynchronous under the hood.** `epoll`, `kqueue`, and `IOCP` are the kernel mechanisms. The event loop is a user-space abstraction over them.
3. **The Actor model scales event-driven across cores.** Each actor is single-threaded; multiple actors run in parallel. Erlang's per-actor isolation is the gold standard for fault tolerance.
4. **Don't block the event loop.** A 1ms computation in a handler delays every other client by 1ms. Offload CPU-intensive work to worker threads or separate processes.
5. **JavaScript's microtask/macrotask distinction is a design choice, not a law.** It ensures Promise callbacks run before the next I/O event, which matches most programmers' mental model.
