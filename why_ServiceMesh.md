Kubernetes out of the box gives you:

| What Kubernetes Gives You | What Kubernetes Does NOT Give You |
|--------------------------|----------------------------------|
| Pod scheduling | Encryption between pods (mTLS) |
| Service discovery | Distributed tracing |
| Load balancing between pods | Canary deployments |
| Health checks and restarts | Automatic retries |
| Scaling | Circuit breaking |

Kubernetes is an orchestrator. It runs your containers and connects them. But it does not manage HOW they talk to each other in a sophisticated way. That is the job of a Service Mesh.

And yes, Istio is the most popular service mesh. Others include:

| Service Mesh | Who Maintains It |
|-------------|-----------------|
| Istio | Google, IBM, Lyft |
| Linkerd | Buoyant |
| Consul Connect | HashiCorp |
| AWS App Mesh | Amazon |

---

## Now Let us Understand gRPC

### Start With the Problem

When two services need to talk to each other, they need to agree on two things:

- How do they send data? (the protocol)
- What format is the data in? (the serialization format)

In the old world, most services used REST over HTTP with JSON. You have definitely seen this:

```
Service A sends:
POST /checkout
{
  "user_id": "12345",
  "cart_items": ["item1", "item2"],
  "payment_method": "card"
}

Service B receives the JSON, parses it, processes it, sends back JSON
```

This works fine. But it has problems at scale inside a microservice architecture:

- JSON is text. It is large and slow to parse
- REST has no strict contract. Service A can send anything and Service B has to handle it
- No built-in streaming support
- No built-in code generation. Every team writes their own HTTP client code

### What is gRPC?

gRPC stands for Google Remote Procedure Call. It was created by Google and open sourced in 2015.

The idea behind RPC is simple. Instead of thinking about sending HTTP requests, you think about calling a function on another service as if it were a local function in your own code.

```
Without RPC:
You think:  "I need to send an HTTP POST to /checkout with this JSON body
             and then parse the JSON response"

With gRPC:
You think:  "I need to call the Checkout() function and get back an OrderResult"
            The network part is hidden from you completely
```

### How gRPC Works

gRPC uses two key technologies together:

The first is Protocol Buffers (Protobuf). This is the data format. Instead of JSON text, data is serialized into a compact binary format. It is much smaller and much faster to parse than JSON.

You define your data structures and functions in a .proto file:

```protobuf
service CheckoutService {
  rpc PlaceOrder(PlaceOrderRequest) returns (PlaceOrderResponse) {}
}

message PlaceOrderRequest {
  string user_id = 1;
  string user_currency = 2;
  Address address = 3;
}
```

The second is HTTP/2. gRPC runs on HTTP/2 instead of HTTP/1.1. HTTP/2 supports multiplexing (multiple requests on one connection), streaming, and header compression.

### gRPC vs REST Comparison

| Feature | REST with JSON | gRPC with Protobuf |
|---------|---------------|-------------------|
| Data format | JSON text | Binary Protobuf |
| Speed | Slower to parse | 5 to 10 times faster |
| Contract | Loose, no strict schema | Strict, defined in .proto file |
| Code generation | Manual | Automatic from .proto file |
| Streaming | Limited | Built-in bidirectional streaming |
| Browser support | Native | Needs a proxy (grpc-web) |
| Human readable | Yes, easy to debug | No, binary format |
| Best for | Public APIs, browser clients | Internal service-to-service calls |

### Why Online Boutique Uses gRPC

The 11 services in Online Boutique talk to each other internally. No browser is involved in service-to-service calls. So gRPC is perfect because:

- It is fast, binary communication between services
- Each service has a strict contract defined in a .proto file
- Code is auto-generated, so each team does not write HTTP client code manually
- It supports multiple programming languages natively, which matters because the services are written in Go, Python, Java, Node.js, and C#

---

## The Full Protocol Landscape

Since you asked about other protocols, here is the complete picture:

### Layer 7 Protocols (Application Level)

| Protocol | Full Name | Format | Best Used For |
|----------|-----------|--------|--------------|
| REST over HTTP/1.1 | Representational State Transfer | JSON or XML | Public APIs, browser clients |
| gRPC over HTTP/2 | Google Remote Procedure Call | Binary Protobuf | Internal microservice calls |
| GraphQL over HTTP | Graph Query Language | JSON | Flexible API queries, mobile apps |
| WebSocket | WebSocket | Any | Real-time bidirectional communication, chat, live feeds |
| SOAP over HTTP | Simple Object Access Protocol | XML | Legacy enterprise systems |
| MQTT | Message Queue Telemetry Transport | Binary | IoT devices, sensors |

### Message Queue Protocols (Async Communication)

| Protocol | Common Tool | Best Used For |
|----------|------------|--------------|
| AMQP | RabbitMQ | Reliable message queuing |
| Kafka Protocol | Apache Kafka | High volume event streaming |
| STOMP | ActiveMQ | Simple text-based messaging |

### How Microservices Choose Their Protocol

```
Are services communicating synchronously (request and wait for response)?
    |
    YES
    |
    Is it a browser or mobile client calling the service?
        YES --> REST with JSON or GraphQL
        NO  --> gRPC (internal service to service)

Are services communicating asynchronously (fire and forget)?
    |
    YES --> Message queue (Kafka or RabbitMQ)
    |
    Examples:
    - Order placed event published to Kafka
    - Email service consumes the event and sends email
    - No waiting, no direct connection between services
```

---

## Putting It All Together for Our Workshop

In Online Boutique:

```
Browser --> frontend service        REST over HTTP (browser cannot speak gRPC natively)

frontend --> all other services     gRPC over HTTP/2 (fast internal calls)

cartservice --> Redis               Redis protocol (key-value store protocol)
```

And when we add Prometheus in Module 7:

```
Prometheus --> all pods             HTTP GET /metrics (simple HTTP scrape)
```

Prometheus uses plain HTTP to collect metrics. It does not use gRPC. The /metrics endpoint returns plain text which is easy, human readable, and universally supported.

