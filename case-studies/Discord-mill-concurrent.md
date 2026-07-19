The transcript from the YouTube video, **"DISCORD SYSTEM DESIGN | HOW DOES DISCORD HANDLE 2 MIIION CONCURRENT VOICE USERS ? | ENGINEERING"** by Sukhad Anand, can be reorganized into the following key points:

### I. The Scaling Problem and Architectural Choice

- Discord's main engineering challenge is scaling to handle **2 million concurrent voice call connections** flawlessly [00:43].
- The system had to be built to support gamers streaming and communicating, a use case that often involves large groups [00:51].
- The two primary models for voice communication are:
    - **Peer-to-Peer (P2P)**: Where clients connect directly.
    - **Client-Server**: Where communication is relayed through a central server [02:24].
- **Peer-to-Peer is rejected for group calls** because it fails when the number of users exceeds approximately 100 [03:12].
    - In a large group call with 'n' people, each client would have to make 'n' connections to listen to everyone [08:28].
    - This keeps thousands of TCP connections open per client, leading to massive bandwidth consumption, high CPU usage, and resource exhaustion [08:47].
- Discord adopts the **Client-Server model** to support thousands of people in a single group call [04:13].

### II. Core Technology: WebRTC and Server Management

- Discord uses **WebRTC** to facilitate communication between peers in a group call [04:50].
- WebRTC is designed for sharing data like video and audio between browsers, creating a persistent streaming connection [05:06].
- Connecting browsers over the internet requires a server to overcome issues like blocked public IP access [06:27].
- The video discusses two types of WebRTC servers:
    - **TURN Server**: Relays a connection so two peers can share data directly [07:05]. Discord **does not** use this, as it poses a security risk (e.g., D-DoS attacks) by exposing the user's IP address [07:26].
    - **WebRTC Media Server (Discord's Choice)**: This server sits between peers to manage and join streams [07:55].
        - The server joins the streams of all speakers and relays **just one single stream** to each client [09:53].
        - This drastically reduces resource utilization, as each client only maintains **one single connection** with the server instead of thousands [09:59].
- The risk of the server being a single point of failure is mitigated by using a **distributed system** where clients can connect to other available servers [10:15].

### III. Discord's Custom Optimizations

Discord implemented several updates and features on top of the base architecture:

- **Updated WebRTC Version**: Discord modified the WebRTC library to prevent **sound attenuation** by operating systems (like Windows) when a call is detected [11:31]. This allows users to hear both the voice chat and the game sound without one being suppressed [11:47].
- **Bandwidth Management**: Connections are closed during periods of silence when no one is speaking [12:15]. The connection is re-initiated by the client when they begin to speak, saving considerable resources [12:34].
- **LXD Framework**: Discord uses LXD, a framework developed on **Erlang**, which is highly effective for concurrent programming and supporting a large number of concurrent operations [12:41].
- **Consistent Hashing**: This technique is used to decide which backend server a client should connect to, particularly when a new person joins or if one server fails [12:56].

The video is available at: DISCORD SYSTEM DESIGN | HOW DOES DISCORD HANDLE 2 MIIION CONCURRENT VOICE USERS ? | ENGINEERING
