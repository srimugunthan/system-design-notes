### Scalability?

- Scalability is the system's ability to handle more load by adding more resources [00:30].
- **Horizontal Scaling** (Preferred in system design) means adding more servers, like hiring more baristas for a cafe [00:47].
- **Vertical Scaling** means upgrading the existing server (like buying a bigger coffee machine), which has limits [00:54].

- 
### Challenges of Scaling

Scaling is complex and comes with challenges [08:14]:

- **Database Bottlenecks:** Slow queries and inefficient indexes can slow the entire system [08:26].
- **Consistency Issues:** In distributed systems, data may not be instantly the same across all nodes, leading to eventual consistency [08:38].
- **Network Latency:** Delays when an app runs across multiple data centers or regions [08:54].
- **Cost Trade-offs:** Scaling always costs more in compute, storage, or bandwidth [09:04].

### 5. Best Practices for Scalable Systems

1. **Design for Failure:** Always build with redundancy and graceful recovery [09:33].
2. **Automate Scaling:** Use autoscaling groups or serverless architectures to handle traffic spikes [09:47].
3. **Go Stateless:** Keep services stateless for simple horizontal scaling [10:00].
4. **Continuous Monitoring:** Track metrics using tools like Grafana or DataDog [10:08].
5. **Start Small:** Don't overengineer; build something that works, monitor it, and grow as users grow [10:19].
