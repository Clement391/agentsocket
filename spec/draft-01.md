# Agent Socket Protocol Specification
**Version:** 0.1.0 (Draft)
**Status:** Experimental / Request for Comment
**Date:** 2024-12-10
**Author:** Kunal Gawde

---

## 1. Abstract
The Agent Socket Protocol (ASP) is a binary-optimized, multiplexed wire protocol designed to enable stateful, real-time communication between AI Agents and Client Interfaces. 

While existing standards (like MCP) rely on stateless HTTP/SSE, ASP utilizes persistent WebSockets with a custom binary framing layer. It is **runtime-agnostic**, designed to support both long-running agent backends (Rust/Python) and ephemeral serverless functions via a relay pattern.

## 2. Terminology
* **Client:** Any consumer interface (Web Frontend, Mobile App, IDE Plugin, CLI, or Voice Client).
* **Agent Runtime:** The compute environment executing the agent logic. This can be:
    * **Stateful:** A long-running server (e.g., Python/FastAPI, Rust/Tokio).
    * **Ephemeral:** A serverless function (e.g., AWS Lambda, Edge Workers).
* **Socket Relay:** An optional edge layer used only in serverless deployments to maintain connection state.

## 3. Protocol Architecture

### 3.1 Connection Lifecycle
The connection is established via a standard WebSocket Upgrade over HTTPS.
`GET /v1/agent-socket HTTP/1.1`
`Upgrade: websocket`
`Sec-WebSocket-Protocol: asp-v1`

### 3.2 Binary Message Frame (Draft)
ASP avoids verbose JSON headers for control messages to maximize throughput. A frame consists of a fixed 4-byte header followed by a variable-length payload.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Ver  |  Type |   Reserved    |          Request ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Payload Length                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Payload (Variable)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
## 4. Message Types (Opcodes)

| Opcode | Name | Direction | Description |
| :--- | :--- | :--- | :--- |
| `0x01` | **HELLO** | C -> S | Client capability advertisement. |
| `0x02` | **INVOKE** | C -> S | Trigger an agent tool or reasoning step. |
| `0x03` | **STREAM** | S -> C | Partial delta of a response (token streaming). |
| `0x04` | **INTERRUPT** | C -> S | Signal to halt current generation/execution. |
| `0x05` | **TOOL_DEF** | S -> C | Async push of a newly discovered tool definition. |
| `0x06` | **ERROR** | Bidirectional | Protocol or Execution error. |

## 5. Modes of Operation
ASP supports two deployment topologies to accommodate user choice:

### 5.1 Direct Mode (Stateful)
* **Best for:** Docker containers, dedicated servers, local dev.
* **Architecture:** The Client connects directly to the Agent Runtime.
* **Implementation:** The Python or Rust SDK handles the WebSocket connection directly in-memory.
* **Latency:** Lowest possible (<10ms).

### 5.2 Relay Mode (Serverless)
* **Best for:** AWS Lambda, Cloudflare Workers, Vercel.
* **Architecture:**
    1.  **Client** connects to a **Socket Relay** (e.g., Redis/Edge/Durable Object).
    2.  **Relay** buffers the frame and invokes the **Agent Function**.
    3.  **Agent Function** (Python/Rust) processes logic and streams chunks back to the Relay.
    4.  **Relay** pushes chunks to the Client.

## 6. Ecosystem Roadmap
The protocol is designed to be language-agnostic.

* **Rust SDK (Core):** High-performance backend implementation for max throughput.
* **Python SDK:** Native bindings for AI/ML workflows (Pydantic integration).
* **Client Libraries:** Reference implementations for React, React Native, and Node.js.

## 7. Security Considerations
* **Authentication:** Performed during the HTTP Upgrade handshake via Bearer Token.
* **Rate Limiting:** Enforced at the Relay or Server level to prevent opcode flooding.

---
*Copyright © 2025 Kunal Gawde. All Rights Reserved.*
