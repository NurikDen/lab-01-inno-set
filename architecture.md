## Product

- **Product name**: Telegram  
- **Website**: [https://telegram.org](https://telegram.org)  
- **Description**: Telegram is a cloud-based instant messaging service that emphasizes speed, security, and cross-platform availability. It supports encrypted chats, large group communications, bots, channels, and seamless media sharing across all user devices.

## Main Components

![Telegram Component Diagram](./docs/diagrams/out/telegram/component-diagram/Component Diagram.svg)

[View PlantUML Source Code](./docs/diagrams/src/telegram/component-diagram.puml)

Selected key components from the diagram:

1. **MTProto Gateway (DC Entry)**: Acts as the entry point for all client connections using Telegram’s custom MTProto protocol. Handles encryption/decryption, routing, and load balancing across data centers.

2. **Auth & Session Service**: Manages user authentication, session validation, device registration, and two-factor authentication logic.

3. **Message Handling Service**: Responsible for processing, storing, and delivering messages. Maintains message order via sequence numbers (PTS) and manages inbox/outbox states.

4. **Media & File Service**: Handles chunked, encrypted uploads and downloads of files and media. Stores file metadata and coordinates with the Distributed File System for persistence.

5. **Notification/Updates Service**: Propagates real-time updates to online clients and triggers push notifications via external services like FCM or APNs based on user settings.

## Data Flow

![Telegram Media Message Flow](./docs/diagrams/out/telegram/sequence-diagram/Sequence Diagram.svg)

[View PlantUML Source Code](./docs/diagrams/src/telegram/sequence-diagram.puml)

**Selected Flow: "1. Media Upload (Encrypted Parts)"**

When Alice sends a photo:
1. The mobile app selects the photo and begins uploading it in encrypted chunks using repeated `saveFilePart` RPC calls.
2. Each chunk is sent through the **MTProto Gateway** to the **Media Service**.
3. The **Media Service** writes each chunk to the **Distributed File System**, validates its checksum, and stores associated metadata (e.g., owner ID, file ID) in the **Sharded Chat DB**.
4. After all parts are uploaded and acknowledged, the app proceeds to send the actual message referencing the uploaded file (`file_id_A`).

*Data exchanged*: Encrypted binary chunks and metadata (file ID, part number) flow from the client → MTProto Gateway → Media Service → Distributed File System and Sharded DB. No unencrypted media is exposed to internal services.

## Deployment

![Telegram Deployment Diagram](./docs/diagrams/out/telegram/deployment-diagram/Deployment Diagram.svg)

[View PlantUML Source Code](./docs/diagrams/src/telegram/deployment-diagram.puml)

Telegram’s infrastructure is deployed globally across multiple data centers:
- **Clients**: Mobile apps (iOS/Android), desktop clients, and web browsers connect via MTProto over TCP, WebSocket, or HTTPS to the nearest **MTProto Gateway** at the edge layer.
- **Compute Layer**: Core services (Auth, Message, Media, etc.) run in containerized pods within Kubernetes-managed clusters for scalability and resilience.
- **Middleware**: Kafka handles event propagation; Redis maintains user state and sequences; custom sharded databases store chat data.
- **Storage**: A **Distributed File System** stores media, while structured data resides in a custom **Sharded Chat DB**.
- **External Services**: Integrations with SMS providers (e.g., Twilio) and push notification platforms (FCM/APNs) occur over secure HTTP/2 or SMPP protocols.

## Assumptions

1. **Media deduplication**: The Distributed File System uses content hashing to avoid storing duplicate files, significantly reducing storage costs for widely shared media.
2. **State synchronization**: The State Cache (Redis) tracks per-user update sequences (PTS) to enable efficient differential sync when clients reconnect.
3. **Secret chats are isolated**: Although not shown in the diagrams, secret chats likely bypass persistent storage entirely, with messages relayed peer-to-peer via the **Secret Chat Relay** without server-side retention.

## Open Questions

1. How does Telegram ensure strong consistency across its **sharded chat databases** during high-throughput scenarios or network partitions?
2. What mechanisms prevent abuse (e.g., spam, DDoS) at the **MTProto Gateway** layer while maintaining low-latency delivery for legitimate traffic?
3. How are encryption keys managed for cloud chats to allow features like search and multi-device sync without compromising user privacy?