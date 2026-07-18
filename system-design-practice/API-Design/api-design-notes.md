



-<img width="1080" height="1350" alt="image" src="goodapis.jp2" />




### Every Type Of API You Must Know Explained!

**(Source: http://www.youtube.com/watch?v=pBASqUbZgkY by Codist )**

---

### 1. REST API (Representational State Transfer)

- **Analogy:** A waiter at a restaurant who takes your order, gets the item, and brings it back to you [00:00].
- **Core Concept:** A simple way for applications to communicate over the internet [00:20].
- **HTTP Methods:** It uses the same HTTP methods you know from web browsing [00:27]:
    - `GET`: To retrieve data (e.g., show all users) [00:27].
    - `POST`: To create something new (e.g., add a new user) [00:35].
    - `PUT`: To update existing data [00:38].
    - `DELETE`: To remove data [00:40].
- **Key Characteristics:**
    - **Stateless:** Each request is completely independent; the server doesn't remember your previous requests, which helps it scale [01:03].
    - **Platform Independent:** Your iPhone app, Android app, or web app can all talk to the same REST API [01:10].
    
    !image.png


    ### SOAP API (Simple Object Access Protocol)

- **Definition:** One of the oldest and most formal ways that systems communicate [01:35].
- **Analogy:** If REST is a casual phone call, SOAP is a formal business contract [01:41].
- **Structure:** Every message must follow strict rules and comes wrapped in **XML** with a specific structure [01:46]: an envelope, a header for metadata, and a body for the request/response [01:56].
- **Key Strengths:**
    - **Protocol Independent:** While commonly used over HTTP/HTTPS, it can also run on SMTP or TCP [02:01].
    - **High Reliability:** Has built-in standards for error handling, security, and transaction support [02:10].
- **Use Cases:** Trusted in industries requiring high reliability and precision, such as banks, healthcare providers, and government systems [02:23].

### 3. gRPC API (g**oogle**RPC)

- **Predecessor:** Based on **RPC (Remote Procedure Call)**, which allows your app to directly call a function on another machine as if it were local [02:54].
- **Analogy:** The "Formula 1 race car of APIs" built for speed and performance [03:32].
- **Data Format:** Uses **Protocol Buffers (Protobuf)**, which compresses data into a compact binary format that is lightning-fast to process (unlike REST's text-based JSON) [03:41].
- **Technology:** Takes advantage of **HTTP/2**, allowing multiple requests to run over a single connection simultaneously [04:01].
- **Performance:** Often 7 to 10 times faster than REST [04:28].
- **Communication Patterns:** Supports four patterns [04:09]: simple request/response, server streaming (live updates), client streaming, and bidirectional streaming.
- **Use Cases:** High-performance systems like Netflix, Uber, and high-frequency trading platforms [04:36].

### 4. GraphQL API (Graph Query Language)

- **Problem Solved:** Addresses overfetching (getting too much data) and underfetching (making multiple API calls) common with REST APIs [04:54].
- **Core Principle:** Allows you to write **one query** asking for *exactly* what data you want (e.g., just the username and email, skipping everything else) [05:22].
- **Mechanism:** One endpoint, one request, perfect data every time [05:31].
- **Killer Feature:** **Real-time Subscriptions** allow your app to listen for live updates automatically [05:32].
- **Features:** It is self-documenting with a built-in playground for testing queries instantly [05:40].
- **Use Cases:** GitHub's entire API, Shopify, and Pinterest [05:46].

### 5. Web Hook API (Reverse API)

- **Traditional API Analogy:** Constantly checking your mailbox to see if something new happened (polling) [06:18].
- **Web Hook Analogy:** The mailman rings your doorbell the moment a letter arrives—it's instant, direct, and efficient [06:31].
- **Core Concept:** Flips the traditional model; instead of your app asking the API, **the API calls you** [06:31].
- **Mechanism:** You set up a **callback URL** in your application, and when an event occurs (e.g., a new payment), the service sends a `POST` request with the event details straight to your URL [06:47].
- **Benefit:** No wasted requests or polling, just real-time updates [06:56].
- **Use Cases:** Powering modern applications like GitHub (on code pushes), Shopify (on new orders), and Slack/Discord bots [07:11].

### 6. WebSockets API

- **Analogy:** Opening a **permanent phone line** between your app and the server [07:34].
- **Connection:** Starts with a handshake to "upgrade" the connection, after which the channel stays open [07:49].
- **Key Feature:** Provides a **persistent two-way communication line** [07:59].
- **Server Push:** Allows the **server to push data** to the client the moment something happens, unlike regular HTTP where the client always initiates [08:06].
- **Use Cases:** Perfect for real-time applications like getting a stock price update, a chat message, or a game event the instant it occurs [08:15].

### 7. WebRTC API (Web Realtime Communication)

- **Definition:** A full framework that enables direct **peer-to-peer communication** between browsers or mobile apps [08:40].
- **The Magic:** The data does not need to flow through a central server [08:44].
- **Use Cases:** Powers video calls, screen sharing, online gaming, and instant file transfers, all inside your browser without extra software [08:47].
- **Mechanism:** It sends video and audio straight from one device to another [09:04].
- **Behind the Scenes:** It automatically handles messy networking details, such as [09:18]:
    - NAT traversal (enabling communication across different networks) [09:24].
    - Negotiating the best audio/video formats [09:31].
    - Adaptive bit rate streaming (adjusting quality based on internet speed) [09:31].
- **Benefit:** Eliminates server bottlenecks, leading to faster communication and smoother real-time experiences [09:43].



-<img width="1080" height="1350" alt="image" src="https://github.com/user-attachments/assets/f0ed2434-d34d-40dc-a918-af0057d39c9b" />

### 2. SOAP API (Simple Object Access Protocol)
