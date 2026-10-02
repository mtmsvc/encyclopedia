# Web Protocols

## HTTP
HTTP (Hypertext Transfer Protocol) is an application-layer, client-server protocol: it defines the format of requests and responses. The actual transport of messages is done by TCP underneath (or QUIC for HTTP/3). It has methods (like GET, POST, PUT, DELETE) and status codes (like 200, 404, 500).

REST (Representational State Transfer) is an architectural style for designing APIs, almost always used over HTTP. It tells how to structure requests and responses around resources. The concepts of a RESTful API:
- **Communication model** - client-server.
- **Resources** - everything is a resource identified by a URL (`/slots`, `/slots/42`), and the HTTP method says what to do with it (GET reads, POST creates, PUT replaces, DELETE removes). URLs are nouns, methods are verbs.
- **Layered system** - the server can be multi-layered (load balancing, proxying, microservice architecture), but the client still sees it as a single server, one big box.
- **Stateless server** - each request is independent: the server does not store any session state, so each new request is treated as the first one. That is why a request must contain everything needed to process it (method, URL, parameters, body, headers).
- **Uniform interface** - consistent request/response formats, naming conventions, status codes, method use cases, etc.
- **Caching** - frequently accessed resources are cached to reduce server load and improve performance (usually only GET responses are cached).

Good practices:
- **Message format** - JSON, XML, etc.
- **Versioning** - if the API endpoints change significantly, add the changes as a new API version. If we just rewrite the old version, clients still using it will break.
- **Documentation** - standard formats for documenting the API (e.g. OpenAPI, Swagger).

## SOAP
SOAP (Simple Object Access Protocol) is a protocol for exchanging structured information between web services. It uses XML as the message format and can be used over any application-layer protocol (HTTP, SMTP, FTP, etc.).

WSDL (Web Services Description Language) is a standard XML-based language for describing web services and their interfaces.

SOAP messages have a strict structure: Envelope, Header, Body.

Usually used in banking, finance and government: strict contracts (WSDL), built-in standards for security and reliable transactions.

## GraphQL
GraphQL is a query language for APIs that lets clients request exactly the data they need and nothing more. The client specifies the data in a query, and the server responds with exactly that data.

Unlike REST with its many endpoints, a GraphQL API usually has one endpoint (e.g. `/graphql`), and requests are typically sent as POST. A schema with types describes everything that can be queried.

Operation types:
- **Query** - retrieves data from the server (like GET in REST).
- **Mutation** - modifies data on the server (like POST, PUT, DELETE in REST).
- **Subscription** - receives data in real time.

## WebSockets
WebSockets is a protocol that provides full-duplex (bidirectional) communication over a single TCP connection. It allows real-time data exchange between a client and a server.

To start a connection, the client sends a standard HTTP request with the `Upgrade` header set to `websocket`. If the server agrees, the connection is established. HTTP is used only for this initial handshake; after that, communication runs directly over TCP.

In HTTP, the server cannot speak unless asked. With WebSockets, the server can send data to the client without the client requesting it.

A server holds many connections (one per client) and can share data it receives from one client with any other client.

Mainly used for online chats, real-time charts, and other real-time data exchange scenarios.

## RPC
RPC (Remote Procedure Call) is a design pattern that allows a client to invoke a procedure on a remote server as if it were a local function call. It is not noun-focused like REST, but verb-focused (functions/actions). In HTTP-based RPC, almost every request is a POST, and the URL is the name of the function we want to call. So REST exposes data, RPC exposes functions.

### gRPC
An RPC framework created by Google.
- Very popular in microservice architectures: microservices communicate with each other via gRPC.
- Uses HTTP/2 instead of HTTP/1.1: faster, sends messages in binary frames instead of text, supports streaming, etc.
- Not used directly by browsers to talk to real users: browsers support HTTP/2, but their JavaScript APIs don't give the low-level control over HTTP/2 that gRPC needs. Browsers need gRPC-Web and a proxy in between.
- Instead of JSON, the message content itself is encoded in a binary format called Protocol Buffers (protobuf).
- Comes with out-of-the-box tooling: code generation for different languages, authentication, streaming (also bidirectional).
- Services and messages are described in protobuf, in `.proto` files. The `protoc` compiler then generates code for different languages.
