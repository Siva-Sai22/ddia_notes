# Reliable, Maintainable, and Scalable System Design

An application may have **functional requirements**(what it should do, how data should be stored, retrieved, searched and processed) and some **non-functional requirements**(security, reliability, scalability, maintainability).

## Reliability

The ability of the service to continue to work correctly,even when things go wrong(faults).

### Faults vs Failures:

- Faults: A component of the system deviating from its expected behavior.
- Failures: When the whole system stops providing the service to the user.

### Faults:

1. Hardware faults: 
    - Redundancy should be present
    - Move towards systems that can tolerate the loss of entire machines

2. Software errors: Mainly caused due to assumptions about the software's environment

3. Human errors:
    - Design systems in a way that minimizes oppurtunities of error
    - Decouple places where people make the mistakes from the places where they can cause failures(provide non-prod sandbox environments)
    - Test thoroughly at all levels
    - Allow quick and easy recovery
    - Detailed and clear monitoring

## Scalability

The ability of the service to cope with increased load.

### Load

Can be best described with load parameters:
- Requests/sec
- Ratio of reads and writes to a database
- Simultaneous active users in a chat room
- Cache hit rate

### Performance

**Latency vs Response Time** : 

- Response time is what the client sees (actual time to process the request + network delays + queueing delays).

- Latency is the duration the request is waiting to be handled.

Response times are generally measured in percentiles(p50, p99). High percentiles of response times are known as *tail latencies*.

These percentiles are often used in SLOs and SLAs.

Small number of slow requests holding up the processing of subsequent requests in known as *head-of-line blocking*.

**Tail latency amplification**: 
Only a small percentage of backend calls are slow but the chance of ggetting a slow call increases if a single user's request requires multiple backend calls. A higher proportion of requests end up being slow.

### Approaches for coping with load

- Vertical and horizontal scaling
- Elastic scaling

## Maintainability

Design software in such a way to minimize pain during maintenance and avoid creating legacy software.

### 1. Operability

Make it easy for operations teams to keep the system running smoothly.

- Monitoring health of the system and restoring it if it goes down
- Tracking down cause of problems
- Keeping software/platforms up to data
- Keeping tabs on how different systems affect each other
- Anticipating future problems and solving them before they occur(capacity planning).
- Establishing good tools for deployment, config management, etc

### 2. Simplicity

Make it easy for new engineers to understand the system, by removing as much complexity as possible.

- **Symptoms**: Explosion of state space, tight coupling of modules, tangled dependencies, etc
- **Abstraction** is the best way to remove it

### 3. Evolvability

Make it easy for engineers to make changes to the system in future. Also known as extensibility, modifiability.