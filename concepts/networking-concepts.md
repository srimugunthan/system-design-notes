### **Networking**

- **IP Address (Internet Protocol Address)** [03:00]
    - Uniquely identifies a device on a network.
- **TCP/IP (Internet Protocol Suite)** [03:03]
    - Includes **TCP** (Transmission Control Protocol) and **UDP** (User Datagram Protocol).
    - **TCP** [03:19] defines the rules for sending data; it breaks files into numbered **packets** [03:33] and ensures missing packets are resent, making it a **reliable protocol** [03:48].
- **Domain Name System (DNS)** [04:03]
    - A decentralized service that translates a domain name (e.g., `neetcode.io`) to its corresponding IP address.
    - A **DNS A Record** (Address) [04:18] is created to map the domain to the server's IP.
- **HTTP (Hypertext Transfer Protocol)** [04:43]
    - An application-level protocol built on top of TCP.
    - It follows the **client-server model** [05:06], where a request includes two parts:
        - **Request Header** [05:06]: Metadata (shipping label).
        - **Request Body** [05:20]: The contents (package).

### Communication and Networking

- **Client-Server Architecture:** The foundation of web applications, where a **Client** (browser, app) sends a request to a **Server**, which processes it and sends a response back.
- **IP Addresses:** Unique identifiers that computers use to locate and communicate with each other on the internet.
- **Domain Name System (DNS):** Maps human-friendly domain names (like `algomaster.io`) to their corresponding IP addresses, which the browser uses to establish a connection.
- **Proxy Server:** Acts as a middleman between your device and the internet, forwarding your request and often hiding your IP address.
- **Reverse Proxy:** Intercepts client requests on the server side and forwards them to the appropriate backend server based on predefined rules.
- **Latency:** The delay in communication, often caused by the physical distance data must travel. Latency is reduced by deploying services across multiple data centers so users connect to the nearest server.
- **HTTP and HTTPS:**
    - **HTTP (Hypertext Transfer Protocol):** The protocol that defines the rules for client and server communication (request/response model).
    - **HTTPS:** The secure version that encrypts all data using SSL or TLS protocol, ensuring security against interception or alteration.
