---
layout: post
title: "Investigation of AMQP1.0 over WebSocket in Python"
# subtitle: "This is the post subtitle."
comments: true
date:   2026-01-09 18:52:21 +0800
categories: 'en'
---

# Investigation of Python Qpid Proton

## Background

We aim to integrate **SAP Event Mesh** with **AMQP 1.0 over WebSocket (amqp10ws)** in Python applications. This transport differs from native **AMQP 1.0 (amqp10)** primarily at the transport layer: the former runs over **WebSocket**, while the latter relies on **raw TCP**.

In the Python ecosystem, there are two commonly referenced AMQP 1.0 client libraries:

- **Apache Qpid Proton**
- **Azure uAMQP**

However, **uAMQP has been officially deprecated and unmaintained since 2025 Q1**, leaving **Qpid Proton** as the only viable long-term option. This raises the question of whether Qpid Proton can support AMQP 1.0 over WebSocket.

---

## Problem Definition

Because **amqp10ws uses a fundamentally different connection establishment mechanism than amqp10**, the first and most critical issue is understanding **how the client connects and authenticates** against SAP Event Mesh.

Specifically:

- Does Python Qpid Proton natively support WebSocket (`ws://`, `wss://`)?
- If not, what would be required to implement AMQP 1.0 over WebSocket in Python?

---

## Analysis

### SAP Reference Implementation

We first examined SAP’s official Node.js implementation:  
`@sap/xb-msg-amqp-v100`.

According to SAP CAP documentation, there are two Event Mesh integration modes:

- `enterprise-messaging`
- `enterprise-messaging-shared`

From the source code, the distinction becomes explicit:

```javascript
"enterprise-messaging": {
  kind: "enterprise-messaging-http",
},
"enterprise-messaging-shared": { // temporary compatibility
  kind: "enterprise-messaging-amqp",
},
```

This indicates that **`enterprise-messaging` uses HTTP-based transport**, while **`enterprise-messaging-shared` uses AMQP**, which is relevant for amqp10ws.

------

### How Does `EnterpriseMessagingShared` Connect to the Broker?

Examining `Client.connect` in `@sap/xb-msg-amqp-v100`, we see that the client explicitly supports:

- Secured / unsecured TCP
- Secured / unsecured WebSocket

```javascript
if (options.tls) {
    tcp.tlsConnect(options, init, fail);
} else if (options.net) {
    tcp.netConnect(options, init, fail);
} else if (options.wss) {
    if(options.oa2) {
        oa2.runGrantFlow(options.oa2,
            (headers) => ws.tlsConnect(options, headers, init, fail),
            fail
        );
    } else {
        ws.tlsConnect(options, {}, init, fail);
    }
} else if (options.ws) {
    if(options.oa2) {
        oa2.runGrantFlow(options.oa2,
            (headers) => ws.netConnect(options, headers, init, fail),
            fail
        );
    } else {
        ws.netConnect(options, {}, init, fail);
    }
} else {
    fail(ErrMsg(EC.CLIENT_MISS_DEST));
}
```

For **SAP Event Mesh**, the execution path is `wss + oa2`, which means the client must first retrieve an **OAuth2 access token** using the **client credentials grant**.

------

### OAuth2 Grant Flow

The grant flow attempts header-based authentication first, falling back to URL parameters if necessary:

```javascript
function runGrantFlow(options, done, failed) {
    if (options.secret) {
        sendAuthViaHeader(options, done, (error) =>
            sendAuthViaParams(options, done, failed, error));
    } else {
        sendAuthViaParams(options, done, (error) =>
            sendAuthViaHeader(options, done, failed, error));
    }
}
```

Once the access token is obtained, it is injected into the **WebSocket upgrade request headers**.

------

### WebSocket Upgrade and Connection Establishment

After authentication, the client initiates an HTTP → WebSocket upgrade:

```javascript
function tlsConnect(options, headers, done, failed) {
    connect(
        https.request(createUpgradeRequest(options.wss, headers)),
        options,
        done,
        failed
    );
}
```

If the upgrade succeeds, the TCP socket is wrapped as a WebSocket connection, and all subsequent traffic consists of **binary AMQP frames over WebSocket**:

```javascript
done(
  new WsConnection(
    options,
    socket,
    new WsReader(options.tune, false),
    new WsWriter(options.tune, socket, true)
  )
);
```

### Transport-Level Difference

This confirms the essential difference between the two transports:

**Native AMQP 1.0**

```
[IP] → [TCP] → [AMQP Frames]
```

**AMQP 1.0 over WebSocket**

```
[HTTP Upgrade] → [WebSocket] → [AMQP Frames]
```

This behavior is fully aligned with the **AMQP WebSocket Binding Specification**:

>The WebSocket Protocol connection MUST be opened as described in Section 4 of the WebSocket specification **[RFC6455]**. The initiating AMQP endpoint (the WebSocket Client) sends a HTTP GET request to the receiving AMQP endpoint (the WebSocket Server) identifying AMQP 1.0 as the subprotocol being used.
>
>...
>
>As per the WebSocket specification **[RFC6455]**, if the Server agrees to the WebSocket upgrade to the requested subprotocol then it MUST respond with an HTTP Status-Line with status code 101 (“Switching Protocols”) and echo the requested subprotocol in the Sec-WebSocket-Protocol HTTP header.

------

## Does Python Qpid Proton Natively Support WebSocket?

The key question is whether Python Qpid Proton can establish such a WebSocket-based connection.

Looking at `Container.connect`, we see that connection handling is delegated to an internal `_Connector` handler:

```python
def _connect(
        self,
        url: Optional[Union[str, Url]] = None,
        urls: Optional[List[str]] = None,
        handler: Optional['Handler'] = None,
        reconnect: Optional[Union[List[Union[float, int]], bool, Backoff]] = None,
        heartbeat: None = None,
        ssl_domain: Optional[SSLDomain] = None,
        **kwargs
) -> Connection:
  conn = self.connection(handler)
	...
  connector = _Connector(conn)
	...
  conn._overrides = connector
	...
  conn.open()
```

The `_Connector` eventually binds the connection to a transport and dispatches a `CONNECTION_BOUND` event, which is handled by `IOHandler.on_connection_bound`.

------

### Transport Binding in Qpid Proton

The relevant implementation in `IOHandler` is:

```python
def on_connection_bound(self, event: Event) -> None:
    ...
    addrs = socket.getaddrinfo(host, port, socket.AF_UNSPEC, socket.SOCK_STREAM)
    sock = IO.connect(addrs[0])
```

And the socket creation logic:

```python
class IO(object):
    @staticmethod
    def connect(addr) -> socket.socket:
        s = socket.socket(addr[0], addr[1], addr[2])
        s.connect(addr[4])
        return s
```

This makes it explicit that:

- Qpid Proton **always uses raw TCP sockets**
- There is **no HTTP handshake**
- There is **no WebSocket framing**
- The URL scheme only affects **TLS configuration (`amqp` vs `amqps`)**

Therefore, **WebSocket (`ws://`, `wss://`) is not natively supported** by Python Qpid Proton.

------

## How Can AMQP 1.0 over WebSocket Be Achieved in Python?

Because **Qpid Proton hardcodes TCP socket creation** via `IOHandler` and `ConnectSelectable`, the default connection mechanism cannot be reused.

To support **AMQP 1.0 over WebSocket** in Python, the following steps are required:

1. **OAuth2 Token Retrieval**
   Authenticate against the `uaa` endpoint from the Event Mesh service key to obtain an `access_token`.
2. **WebSocket Connection Establishment**
   Use a WebSocket client library (e.g. `websocket-client` or `websockets`) to connect to the `wss://` endpoint, injecting the `Authorization: Bearer <token>` header.
3. **Manual Proton Transport Binding**
   Bind the WebSocket’s binary send/receive stream to a `proton.Transport` instance, effectively replacing Proton’s internal TCP-based I/O layer.

This approach requires **bypassing or re-implementing Proton’s I/O layer**, which is non-trivial and explains why existing support is limited.

------

## References

1. Messaging Protocols and Libraries
    https://help.sap.com/docs/event-mesh/event-mesh/messaging-protocols-and-libraries
2. `@sap/xb-msg-amqp-v100`
    https://www.npmjs.com/package/@sap/xb-msg-amqp-v100
3. SAP Event Mesh (Shared)
    https://cap.cloud.sap/docs/node.js/messaging#event-mesh-shared
4. AMQP WebSocket Binding Specification
    https://docs.oasis-open.org/amqp-bindmap/amqp-wsb/v1.0/cs01/amqp-wsb-v1.0-cs01.html

