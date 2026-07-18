### **Load Balancing**

Load balancing distributes user requests across multiple servers to handle scale and ensure high availability [02:02:30].

- **Necessity:** One server is not enough to handle a service at scale, and if a single server crashes, customers lose access [02:02:04].
- **Algorithms for Distribution:**
    - **Round Robin:** Requests are sent one after another to each server in a rotating sequence. It is simple but doesn't account for uneven workloads [02:02:37].
    - **Least Connections:** Traffic is routed to the server currently handling the fewest active requests. This is more adaptive but requires tracking active requests per server [02:02:46].
    - **Hash-Based:** The request is routed based on a key (like a user ID). This is great for keeping related traffic together but can create "hotspots" if keys are not well-distributed [02:02:51].
- **Load Balancer Layers:**
    - **Layer 4:** Looks at network information, such as IP address and port [03:03:04].
    - **Layer 7:** Can inspect deeper information like HTTP headers and URLs. This offers more control but adds additional overhead [03:03:06].
