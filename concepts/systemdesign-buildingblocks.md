### Building Blocks of a Scalable System

A typical scalable system logically includes:

- **Client Layer:** Web or mobile app sending requests [01:24].
- **CDN (Content Delivery Network):** Delivers static content (images, CSS) close to the user [01:17].
- **Load Balancer:** Distributes incoming traffic evenly across servers [01:33].
- **Application Servers:** The logic layer, usually restricted to microservices [01:37].
- **Cache Layer:** Fast memory to reduce database hits [01:40].
- **Database Layer:** Where all the data lives [01:43].
- **Message Queues:** To handle asynchronous jobs and traffic spikes [01:46].
- **Monitoring and Logging:** Essential because "what you can't measure you can't scale" [01:49].
