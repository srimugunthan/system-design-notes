This talk by Paul Bakker (2026 Edition) provides a deep dive into how Netflix leverages Java to power its massive streaming ecosystem, highlighting their architectural choices, migration strategies, and recent innovations.

### **The Role of Java at Netflix**
Java is the backbone of the Netflix backend, providing a balance between runtime performance, developer productivity, and maintainability [[08:24](http://www.youtube.com/watch?v=ucJTPda_zx0&t=504)].
* **Streaming Services:** High-performance, multi-region microservices handling millions of requests per second [[03:15](http://www.youtube.com/watch?v=ucJTPda_zx0&t=195)].
* **Studio Software:** More traditional enterprise applications used for movie production [[03:42](http://www.youtube.com/watch?v=ucJTPda_zx0&t=222)].
* **Open Connect:** The physical hardware and management software that handles the actual video bit streaming [[07:05](http://www.youtube.com/watch?v=ucJTPda_zx0&t=425)].


### **The Tech Stack**
* **Spring Boot:** Standardized as the "Paved Road" for Netflix developers, providing out-of-the-box integration with security, configuration, and observability systems [[11:58](http://www.youtube.com/watch?v=ucJTPda_zx0&t=718)].
* **GraphQL & gRPC:** Netflix uses a federated GraphQL architecture for client-to-server discovery [[05:07](http://www.youtube.com/watch?v=ucJTPda_zx0&t=307)], while gRPC is reserved for high-performance server-to-server communication [[27:55](http://www.youtube.com/watch?v=ucJTPda_zx0&t=1675)].
* **DGS Framework:** Netflix open-sourced the Domain Graph Service (DGS) framework to simplify GraphQL development in Java [[28:49](http://www.youtube.com/watch?v=ucJTPda_zx0&t=1729)].

### **JVM Innovations**
Netflix remains at the cutting edge of JVM features to optimize their massive infrastructure.
* **Generational ZGC:** Now the default garbage collector [[35:11](http://www.youtube.com/watch?v=ucJTPda_zx0&t=2111)]. It eliminated long "Stop the World" pause times, significantly reducing IPC client errors and retries [[33:56](http://www.youtube.com/watch?v=ucJTPda_zx0&t=2036)].
* **Virtual Threads:** After a "false start" due to thread pinning, Netflix is rolling out virtual threads again with JDK 25 [[35:25](http://www.youtube.com/watch?v=ucJTPda_zx0&t=2125)].
* **Context Propagation:** To handle thread locals with virtual threads, Netflix uses specialized thread factories and libraries like Micrometer to copy security and tracing contexts [[39:17](http://www.youtube.com/watch?v=ucJTPda_zx0&t=2357)].


### **AI and Agentic Workflows**
Netflix is integrating Generative AI into its Java microservices using **Spring AI** [[45:14](http://www.youtube.com/watch?v=ucJTPda_zx0&t=2714)].
* **Agentic Workflows:** Instead of "black box" agents, they build workflows where the developer maintains control, using LLMs for specific tasks like routing or summarizing [[42:02](http://www.youtube.com/watch?v=ucJTPda_zx0&t=2522)].
* **Developer Productivity:** They built a Java service that analyzes profiling data and uses an LLM to give actionable recommendations to developers on how to fix slow Spring bean initialization [[46:46](http://www.youtube.com/watch?v=ucJTPda_zx0&t=2806)].

### **Migration Strategies**
Managing thousands of services requires automated migration tools:
* **Java X to Jakarta:** They built a Gradle transform that uses bytecode manipulation to rewrite library code during artifact resolution, solving the "chicken and egg" problem of library compatibility [[19:33](http://www.youtube.com/watch?v=ucJTPda_zx0&t=1173)].
* **AI-Driven Upgrades:** For Spring Boot 4, they are experimenting with an agentic workflow using **Claude** to automate complex code migrations that are difficult to handle with deterministic rules like OpenRewrite [[22:18](http://www.youtube.com/watch?v=ucJTPda_zx0&t=1338)].

### **Why Not Other Languages?**
While Netflix uses JavaScript for UIs and Python for Machine Learning [[10:43](http://www.youtube.com/watch?v=ucJTPda_zx0&t=643)], Java remains the primary choice for backend work because it is faster than Python and more productive/maintainable than Rust or C++ for their specific scale [[09:41](http://www.youtube.com/watch?v=ucJTPda_zx0&t=581)].


http://googleusercontent.com/youtube_content/1
