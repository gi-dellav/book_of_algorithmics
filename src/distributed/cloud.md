# Cloud Computing

Cloud computing is the infrastructure layer beneath distributed systems. Instead of buying servers, you rent them. Instead of managing switches, you configure virtual networks. Instead of forecasting capacity, you auto-scale. The cloud doesn't change the algorithms — it changes the economics, the failure model, and the operational complexity.

## The Abstraction Layers

### IaaS (Infrastructure as a Service)

Virtual machines, block storage, and virtual networks. You manage the OS, the runtime, and your application. The cloud provider manages the hypervisor, the physical servers, and the network fabric.

Example: AWS EC2. You launch a `c5.4xlarge` instance (16 vCPUs, 32 GB RAM), install your software, and pay per second. When you're done, terminate it. No hardware procurement, no data center visits, no switch configuration.

**When to use IaaS**: you need full control over the environment (custom kernels, GPU drivers, specific OS versions), or you're migrating an existing application that runs directly on a server.

### Containers and Kubernetes

Containers package an application and its dependencies into a portable image. Unlike VMs, containers share the host kernel — they're lighter (MB vs. GB), start faster (seconds vs. minutes), and have negligible CPU overhead.

Kubernetes orchestrates containers across a cluster:
- **Pods**: groups of containers that share a network namespace and storage. The unit of scheduling.
- **Deployments**: declarative specification of desired state ("run 3 replicas of my app"). Kubernetes reconciles actual state to match.
- **Services**: stable network endpoints for pods (which come and go). Load balancing across pod replicas.
- **Auto-scaling**: increase replicas based on CPU/memory/custom metrics.

Kubernetes is the "distributed OS" for the cloud. It handles scheduling, networking, storage, and health checking. You specify what you want; Kubernetes makes it happen.

### Serverless (FaaS: Function as a Service)

Functions are the smallest unit of computation. You write a function; the cloud provider runs it when triggered (HTTP request, queue message, cron schedule). No servers, no containers, no scaling configuration.

```python
# AWS Lambda function
def lambda_handler(event, context):
    # event = {'key': 'value'}
    result = process(event['key'])
    return {'statusCode': 200, 'body': result}
```

Pricing: per invocation ($0.20 per million) + per GB-second of memory ($0.000016667 per GB-second). A function using 256 MB running for 100 ms: 0.000025 GB-seconds → effectively free for low-volume use.

**When to use serverless**: event-driven workloads (file upload triggers, API backends with sporadic traffic), glue code between cloud services, and low-volume applications where running a 24/7 server is wasteful.

**When not to use serverless**: long-running computations (Lambda has a 15-minute timeout), high-throughput streaming (the per-invocation overhead is too high), and applications requiring persistent connections (WebSockets, though some platforms now support them).

## The Economics

| Model | Cost Structure | Best For |
|-------|---------------|----------|
| On-premises | Capital expense (servers) + operational (power, cooling, staff) | Predictable, steady workloads |
| IaaS (VMs) | Per second/minute/hour | Variable workloads, batch processing |
| Kubernetes | Per VM (the underlying nodes) | Microservices, multi-service deployments |
| Serverless | Per invocation + per GB-second | Sporadic, low-volume, event-driven |
| Spot instances | 60-90% cheaper, but can be terminated at any time | Fault-tolerant batch jobs |

Spot instances (AWS) / preemptible VMs (GCP) are the cheapest compute: unused capacity sold at a discount, with the caveat that your instance can be terminated with 2 minutes' notice. For fault-tolerant batch jobs (MapReduce, ML training with checkpointing), spot instances cut costs by 70%. For serving live traffic, use on-demand or reserved instances.

## Cloud-Native Design Patterns

### Stateless Services

Each request is independent. No session state stored on the server. This enables horizontal scaling: add more replicas, and any replica can handle any request. State is stored in external services (databases, caches, object storage).

### Circuit Breaker

If a downstream service is failing, stop calling it. After a timeout, try again. This prevents cascading failures: if service A calls B calls C, and C is slow, B's thread pool fills up waiting for C, then A's fills up waiting for B. The circuit breaker breaks the chain.

```python
# Simplified circuit breaker
class CircuitBreaker:
    def __init__(self, timeout=60, failure_threshold=5):
        self.failure_count = 0
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
        self.last_failure_time = 0
    
    def call(self, fn):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"  # Try again
            else:
                raise CircuitBreakerOpen()
        
        try:
            result = fn()
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
            self.failure_count = 0
            return result
        except Exception:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise
```

### Bulkhead

Isolate resources so that a failure in one component doesn't exhaust resources for others. In Kubernetes: set resource limits per container. In thread pools: use separate pools for different operations.

### Retry with Backoff

Transient failures (network blips, service restarts) should be retried, but with exponential backoff to avoid the "thundering herd" problem (all clients retry simultaneously):

```
Attempt 1: immediate
Attempt 2: wait 1 second
Attempt 3: wait 2 seconds
Attempt 4: wait 4 seconds
Attempt 5: wait 8 seconds
... up to max_backoff (e.g., 60 seconds)
```

Add jitter (±25% of the delay) to spread retries across time.

## The Cloud and Distributed Algorithms

The cloud doesn't change the algorithms from earlier in this chapter, but it changes the assumptions:

| Assumption | On-Premises | Cloud |
|-----------|------------|-------|
| Node reliability | High (enterprise hardware) | Medium (commodity hardware, shared) |
| Network reliability | High (dedicated switches) | Medium (shared fabric, noisy neighbors) |
| Latency | Predictable (known topology) | Variable (multi-tenant, unknown topology) |
| Bandwidth | Guaranteed (dedicated links) | Variable (burst credits, throttling) |
| Scaling | Slow (procurement) | Fast (API calls) |
| Cost model | Capital + operational | Per-use |

The takeaway: cloud-native distributed algorithms must be more tolerant of variance (latency spikes, bandwidth fluctuations, node failures) but can be more elastic (scale up for a big job, scale down when done). The algorithms are the same; the engineering is different.

## Key Lessons

1. **The cloud is an operational model, not an algorithmic one.** Moving from on-premises to cloud doesn't change the algorithms — it changes deployment, scaling, and failure handling.
2. **Spot instances are the cheapest compute, if your algorithm tolerates preemption.** Checkpointing + spot instances = 70% cost reduction for batch jobs.
3. **Serverless is for glue code and sporadic workloads, not for high-throughput processing.** The per-invocation overhead (~1 ms cold start, ~0.1 ms warm) adds up.
4. **Cloud-native patterns (circuit breaker, retry with backoff, bulkhead) are about graceful degradation.** Assume everything can fail, and design for partial functionality rather than binary up/down.
5. **Kubernetes is the standard for container orchestration.** It's complex, but it solves the hard problems (scheduling, service discovery, rolling updates, auto-scaling) that you'd otherwise build yourself.
