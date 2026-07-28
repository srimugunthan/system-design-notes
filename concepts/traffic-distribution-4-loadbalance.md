
## 2. The Role of a Load Balancer

A **Load Balancer (LB)** acts as a digital traffic cop sitting between your users and your fleet of servers.

Instead of users connecting directly to an application server, all incoming user traffic hits the Load Balancer's IP address. The LB then decides which backend server instance should handle each incoming request.

* **Health Monitoring:** LBs constantly ping your backend servers (health checks). If a server crashes or stops responding, the LB automatically routes traffic around it until it recovers.

---

## 3. Traffic Distribution Strategies

How does the Load Balancer choose which server gets the next request?

### A. Round-Robin

The simplest approach. The LB cycles through the servers sequentially in a loop: Request 1 goes to Server A, Request 2 to Server B, Request 3 to Server C, Request 4 back to Server A.

* **Best for:** Homogeneous environments where all servers have identical hardware specs and incoming requests take roughly equal processing power.

### B. Weighted Round-Robin / Health-Based

* **Weighted:** If Server A is twice as powerful as Server B, you assign Server A a higher "weight." The LB will route 2 requests to Server A for every 1 request sent to Server B.
* **Health-Based / Least Connections:** The LB tracks active loads in real-time and sends the next request to whichever server currently has the fewest active connections or lowest CPU utilization.

### C. Sticky Sessions (Session Affinity)

The Load Balancer uses a cookie or IP address mapping to ensure that **all requests from User X always go to Server A**, for the duration of their browsing session.

* **Why use it?** If an legacy application was *not* properly decoupled in Phase 1 (i.e., user session data is still stored in local server RAM), sticky sessions prevent the user from being logged out when navigating between pages.
* **The Drawback:** It breaks true load balancing. If Server A gets assigned 50 heavy power-users, Server A gets overloaded while Server B sits idle. It also makes server restarts difficult without disrupting those pinned users. *(This is why modern systems prefer stateless servers over sticky sessions).*
