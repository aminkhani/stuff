## 💡 What is gRPC?

- **gRPC** is a high‑performance, open‑source RPC (Remote Procedure Call) framework originally developed by Google.    
- It uses **Protocol Buffers** (protobufs) as its Interface Definition Language (IDL) and for efficient binary serialization.
- Communication happens over HTTP/2, giving you features like multiplexing, streaming, and built‑in flow control.
---
## 🤔 Why use gRPC?

- **Performance**: Binary protobuf payloads are smaller and faster to serialize/deserialize than JSON.
- **Strongly‑typed APIs**: Schemas (proto files) guarantee both client and server agree on message formats.
- **Bi‑directional streaming**: Easily support server‐push or full duplex streams.
- **Multi‑language support**: Auto‑generated stubs for dozens of languages.
- **Built‑in code generation**: Avoid writing boilerplate serialization or HTTP plumbing.
---
## 📆 When to use gRPC?

- **Microservices** that need efficient inter‐service communication.
- **Mobile and IoT** clients where bandwidth and latency matter.
- **Polyglot environments**—when you have services in multiple languages.
- **Streaming scenarios**—real‑time data feeds, chat, telemetry, etc.
---
## 🛠️ How to use gRPC in Python

### 1. Install dependencies
```bash
pip install grpcio grpcio-tools
```
### 2. Define your service in a `.proto` file
```proto
// greeter.proto
syntax = "proto3";

package greeter;

service Greeter {
  // Unary RPC
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```
### 3. Generate Python code
```bash
python -m grpc_tools.protoc \
  --python_out=. \
  --grpc_python_out=. \
  --proto_path=. \
  greeter.proto
```
This creates `greeter_pb2.py` and `greeter_pb2_grpc.py`.
### 4. Implement the Server
```python
# server.py
from concurrent import futures
import grpc
import greeter_pb2, greeter_pb2_grpc

class GreeterServicer(greeter_pb2_grpc.GreeterServicer):
    def SayHello(self, request, context):
        name = request.name
        print(f"📩 Received request for: {name}")
        reply = greeter_pb2.HelloReply(message=f"Hello, {name}!")
        print(f"📤 Sending response: {reply.message}")
        return reply

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    greeter_pb2_grpc.add_GreeterServicer_to_server(GreeterServicer(), server)
    port = 50051
    server.add_insecure_port(f"[::]:{port}")
    server.start()
    print(f"🟢 Server started on port {port}")
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
```