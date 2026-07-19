The video "How NGINX Handles MILLIONS of Requests With Just 1 Process 🔥" explains the architecture of NGINX, focusing on why it is so fast and scalable compared to traditional web servers.
https://www.youtube.com/watch?v=I6dpN0geIb4

Here is the transcript reorganized into key points:

### **1. The Problem with Traditional Web Servers (Apache)**

- **Model:** Traditional servers like Apache use a **thread-per-request model** [00:50]. This means a new thread is spawned for every incoming user connection [00:57].
- **The Scaling Issue:** As traffic scales, this leads to:
    - High memory usage [01:22].
    - Frequent **context switching** [01:22].
    - Server slowdowns or crashes when handling thousands of concurrent connections [01:26]. This model is not designed to scale well for massive concurrency [01:40].

### **2. NGINX's Event-Driven Architecture**

- **Core Approach:** NGINX takes a completely different, **event-driven, non-blocking** approach built for concurrency [00:14]. It does not spin up a new thread for every request [01:54].
- **Process Structure:** NGINX runs a small number of processes:
    - **Master Process:** Manages configuration, spawns worker processes, and handles graceful reloads of updates [02:10].
    - **Worker Processes:** The "real workhorses," usually configured to run **one per CPU core** [01:59]. Each worker is an independent, separate process with its own memory [05:10].
    - **Cache Manager/Loader:** Optional processes that handle caching for static content [02:30].

### **3. How a Single Worker Handles Thousands of Connections**

- **Event Loop:** Each worker process uses an **event loop** to switch between active tasks [02:25].
- **Non-Blocking I/O:** When a connection is waiting on Input/Output (I/O)—like reading a file or fetching from a database—the worker **doesn't block**. Instead, it places that connection aside and moves on to the next ready connection [03:46].
- **Efficient System Calls (e.g., epoll):** NGINX uses efficient system calls:
    - `epoll` for Linux [05:55].
    - `kqueue` (`k_Q`) for BSD or Mac OS [05:55].
    - These calls allow the event loop to monitor thousands of sockets at once without manually looping through them [06:58]. The system call only **notifies** the worker when a specific connection is ready to read or write data [07:11].
- **The Result:** By processing only the connections that are ready ("picking up the ringing phone lines"), NGINX remains completely non-blocking and ultra-responsive, cycling through thousands of sockets per second even under massive load [08:34].

### **4. NGINX vs. Node.js (Concurrency Model)**

| **Feature** | **NGINX** | **Node.js** |
| --- | --- | --- |
| **Concurrency Unit** | Master process spawns multiple **worker processes** (one per CPU core) [05:10]. | Starts with a single **JavaScript thread** [04:50]. |
| **Memory/Isolation** | Each worker is a separate, isolated process with its own memory, resulting in more stability [05:47]. | Threads share memory within the same process, which can lead to race conditions [05:40]. |
| **I/O Offloading** | Uses efficient system calls (`epoll`/`kqueue`) to monitor thousands of connections using a single loop [05:55]. | Uses **libUV** to offload I/O tasks to a behind-the-scenes thread pool [04:54]. |

You can view the video here: https://www.youtube.com/watch?v=I6dpN0geIb4
