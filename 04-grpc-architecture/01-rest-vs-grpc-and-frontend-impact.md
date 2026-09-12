# REST vs gRPC — What Actually Changes, and What It Means for the Frontend

This note answers four things in order: what gRPC fundamentally changes vs REST, what changes on the backend if we move our Go API to gRPC, whether the frontend (React) can call gRPC the same way it calls REST, and — if the frontend wants to keep making normal-looking calls but just swap JSON for protobuf — what the backend needs to do differently for *that* specific setup.

---

## 1. REST vs gRPC — the fundamental differences

| | REST | gRPC |
|---|---|---|
| **Transport** | HTTP/1.1 (usually) | HTTP/2 (required) |
| **Message format** | JSON — human-readable text | Protobuf — compact binary |
| **API contract** | Informal (docs, OpenAPI/Swagger) — nothing enforces the client and server agree | `.proto` file — a formal schema that *generates* client and server code, so mismatches are caught at compile time |
| **Communication style** | Request → response, one at a time | Request → response, **plus** client streaming, server streaming, and bidirectional streaming |
| **Browser support** | Native — `fetch()` just works | Not directly supported by browsers (explained in section 3) |
| **Typical use case** | Public APIs, browser clients, anything that benefits from being human-readable/debuggable | Internal service-to-service calls, mobile clients, high-throughput systems, streaming use cases |

**The two changes that matter most for everything downstream:**
1. **Binary instead of text.** You can't just `curl` a gRPC endpoint and read the response like you could with JSON — it's compact binary bytes that only make sense once decoded using the `.proto` schema.
2. **A schema-first contract.** With REST, your `User` struct in Go and your frontend's expectations of the JSON shape are two separate things that *happen* to agree (or drift apart, causing bugs). With gRPC, both sides generate code from the same `.proto` file — so the contract is enforced by tooling, not convention.

---

## 2. What changes on the backend, concretely

Going back to our Go REST server, here's the same `User` resource, gRPC-style.

**Before (REST) — the contract lived only in your Go struct and documentation:**
```go
type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}
```

**After (gRPC) — the contract lives in a `.proto` file first:**
```proto
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc CreateUser (CreateUserRequest) returns (User);
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
message GetUserRequest { int32 id = 1; }
message CreateUserRequest { string name = 1; string email = 2; }
```

You then run a code generator (`protoc`) which produces Go types and interfaces for you — you no longer hand-write the `User` struct; it's generated from the `.proto` file.

**Your HTTP handlers turn into RPC method implementations:**
```go
// Before: an HTTP handler function
func createUserHandler(w http.ResponseWriter, r *http.Request) { ... }

// After: a method implementing the generated UserServiceServer interface
func (s *server) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
	newUser := &pb.User{
		Id:    int32(len(users) + 1),
		Name:  req.Name,
		Email: req.Email,
	}
	users = append(users, newUser)
	return newUser, nil
}
```

**What disappears entirely:**
- Manual `json.NewDecoder(...).Decode(...)` and `json.NewEncoder(...).Encode(...)` — serialization is handled for you by the generated code.
- Manual routing (`switch r.Method`, `r.URL.Path` parsing for path params) — the gRPC framework routes calls to the right method based on the service definition, not URL parsing.
- Manually setting `Content-Type` headers — gRPC always uses a fixed content type (`application/grpc`) since the format is standardized.
- Status codes like `200`/`404`/`422` — gRPC has its own status code set (`OK`, `NOT_FOUND`, `INVALID_ARGUMENT`, etc.), returned via an `error` return value instead of `w.WriteHeader(...)`.

**What stays conceptually the same:**
- You still have "read a user," "create a user" as distinct operations — just expressed as RPC methods instead of HTTP verbs + URLs.
- Auth still happens via metadata attached to the call (gRPC's equivalent of headers), often still a bearer token.
- You still run this inside a container, behind a load balancer, on ECS — as covered in the deployment note (with the load-balancing and health-check changes discussed there).

---

## 3. Can the frontend just call gRPC normally? — No, and here's why

**The short answer: browsers cannot make native gRPC calls.** This isn't a tooling gap that'll get fixed later — it's a fundamental limitation:

- gRPC relies on **HTTP/2 trailers** to send the final status code and metadata *after* the response body streams. Browser APIs (`fetch`, `XMLHttpRequest`) don't expose HTTP/2 trailers to JavaScript at all — there's no way for your React code to read them, even though the browser's network stack technically supports HTTP/2.
- gRPC also relies on the ability to control low-level framing of HTTP/2 streams (for streaming RPCs), which browsers don't expose to application code either.

So if you point `fetch()` directly at a real gRPC endpoint, it will not work correctly — you won't be able to properly read the response, especially the status/trailers, and streaming RPCs won't function at all.

### What the frontend needs instead: gRPC-Web

**gRPC-Web** is a variant protocol + a client library specifically built to work around this. It requires two changes:

1. **A proxy layer between the browser and your gRPC backend.** Typically **Envoy** (with its grpc-web filter) or a similar proxy sits in front of your Go gRPC server, translating gRPC-Web requests coming from the browser into real gRPC calls to your backend, and translating the real gRPC response (including those trailers) back into something the grpc-web client can read.

```
React (grpc-web client) → Envoy (grpc-web proxy) → Go gRPC server
```

2. **Frontend code generated from the same `.proto` file**, using the grpc-web code generator, instead of hand-writing `fetch()` calls:

```js
import { UserServiceClient } from "./generated/user_grpc_web_pb";
import { CreateUserRequest } from "./generated/user_pb";

const client = new UserServiceClient("https://api.yourapp.com");

const request = new CreateUserRequest();
request.setName("Alex");
request.setEmail("alex@example.com");

client.createUser(request, {}, (err, response) => {
  if (err) {
    console.error(err.message);
    return;
  }
  console.log(response.getId(), response.getName());
});
```

Notice this looks nothing like `fetch()` — there's no manual header-setting, no `JSON.stringify`, no checking `res.status`. You call a generated method, pass a generated request object, and get a generated response object back (or an error). The client library and Envoy proxy handle everything below that.

**Practical implication:** adopting real gRPC end-to-end means your frontend team adopts new tooling (protoc + grpc-web generator, a different client library, a build step to regenerate stubs whenever `.proto` files change), and your infra team adds a proxy layer that didn't exist before. This is a real, non-trivial shift — not just "swap the URL."

---

## 4. If the frontend wants to keep making REST-style calls, but use protobuf instead of JSON

This is a genuinely different (and simpler) setup than "real gRPC." What you're describing is often called **"protobuf over HTTP"** or **"protobuf as a REST body format"** — you keep the REST architecture (HTTP/1.1, verbs, URLs, status codes) entirely intact, and just change the *serialization format* of the body from JSON to protobuf binary. You are **not** using gRPC's transport, streaming, or HTTP/2 requirement at all — this avoids every browser limitation from section 3, because it's still just a normal HTTP request/response with `fetch()`.

### What the backend needs to change

**1. Still define your data shape in a `.proto` file** (this part doesn't change vs full gRPC — you want the schema/codegen benefits):
```proto
syntax = "proto3";

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```
Run `protoc` to generate the Go struct (`pb.User`) — but note there's no `service` block needed here, since we're not using gRPC's RPC mechanism, just its message format.

**2. Change `Content-Type` handling.** Instead of `application/json`, both sides agree on a protobuf-specific content type:
```
Content-Type: application/x-protobuf
```

**3. Replace `encoding/json` calls with protobuf's `proto.Marshal` / `proto.Unmarshal`:**

```go
import "google.golang.org/protobuf/proto"

func createUserHandler(w http.ResponseWriter, r *http.Request) {
	// Read the raw bytes from the body (instead of json.NewDecoder)
	body, err := io.ReadAll(r.Body)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	var newUser pb.User
	if err := proto.Unmarshal(body, &newUser); err != nil { // instead of json.Unmarshal
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	newUser.Id = int32(len(users) + 1)
	users = append(users, &newUser)

	respBytes, _ := proto.Marshal(&newUser) // instead of json.Marshal
	w.Header().Set("Content-Type", "application/x-protobuf")
	w.WriteHeader(http.StatusCreated)
	w.Write(respBytes)
}
```

**Everything else stays exactly as it was in the REST guide** — the URL routing (`/users`, `/users/3`), the HTTP verbs (GET/POST/PUT/PATCH/DELETE), the status codes (200/201/404/422), auth via the `Authorization` header, CORS, path/query params — none of that changes. You've only swapped the body's encoding.

### What the frontend needs to change (for this hybrid approach)

Frontend keeps using `fetch()` exactly as before — no grpc-web, no Envoy proxy, no HTTP/2 requirement. The only changes are how the body is built and read:

```js
import { User } from "./generated/user_pb"; // generated by protoc for JS

async function createUser(name, email) {
  const newUser = new User();
  newUser.setName(name);
  newUser.setEmail(email);

  const res = await fetch("http://localhost:8080/users", {
    method: "POST", // still a normal REST verb
    headers: {
      "Content-Type": "application/x-protobuf", // instead of application/json
      "Authorization": "Bearer secret123",       // unchanged
    },
    body: newUser.serializeBinary(), // instead of JSON.stringify(...)
  });

  const buffer = await res.arrayBuffer(); // instead of res.json()
  const createdUser = User.deserializeBinary(new Uint8Array(buffer));

  console.log(res.status, createdUser.getId(), createdUser.getName());
  return createdUser;
}
```

The differences from the original REST example, side by side:

| | JSON REST (original) | Protobuf-over-REST (hybrid) |
|---|---|---|
| Build request body | `JSON.stringify({ name, email })` | `new User(); user.setName(...); user.serializeBinary()` |
| `Content-Type` | `application/json` | `application/x-protobuf` |
| Read response body | `await res.json()` | `await res.arrayBuffer()` then `User.deserializeBinary(...)` |
| URL, method, status codes, headers, auth | unchanged | unchanged |
| Needs Envoy/grpc-web proxy? | No | **No** |
| Needs HTTP/2? | No | No — plain HTTP/1.1 still works fine |

### Why you'd choose this hybrid approach

- You get protobuf's benefits — smaller payloads, faster serialization, a strict schema that catches mismatches — without touching your existing REST infrastructure, load balancer config, or browser compatibility.
- You keep debugging tools like Postman/curl mostly usable (though you'd need a protobuf-aware tool or the `.proto` schema on hand to decode responses, since it's no longer human-readable JSON).
- You avoid the biggest gRPC costs from section 3: no Envoy/grpc-web proxy layer, no HTTP/2-aware load balancer reconfiguration, no new client library for the frontend team to learn.

### The full backend structure for this hybrid approach

Here's what an actual project layout looks like, and how it differs from the plain JSON REST version.

**Project structure:**
```
my-rest-api/
├── proto/
│   └── user.proto          # NEW — defines the message schema
├── pb/
│   └── user.pb.go          # NEW — generated by protoc, don't hand-edit
├── main.go                 # CHANGED — encode/decode logic swapped to protobuf
├── go.mod
└── go.sum
```

Compare this to the plain JSON version, which was just:
```
my-rest-api/
├── main.go                 # User struct + handlers, all in one file
├── go.mod
└── go.sum
```

The key structural change: **your data shape moves out of your handler code and into a `.proto` file**, and a generated file sits between them. You never hand-write the `User` struct anymore — it's generated.

---

**`proto/user.proto`** — the schema, single source of truth for the shape of `User`:
```proto
syntax = "proto3";
package pb;
option go_package = "my-rest-api/pb";

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

message UserList {
  repeated User users = 1;
}

message ErrorResponse {
  string error = 1;
}
```

Note there's no `service` block (no RPC methods defined) — because we're not using gRPC's calling convention at all, just its message/schema format. This is the concrete marker that distinguishes this hybrid from full gRPC.

Generate the Go code from it:
```bash
protoc --go_out=. --go_opt=paths=source_relative proto/user.proto
```
This produces `pb/user.pb.go`, containing a generated `pb.User` struct (with `GetId()`, `GetName()`, etc. accessor methods) and all the binary marshal/unmarshal machinery — you never write or edit this file by hand; you regenerate it whenever `user.proto` changes.

---

**`main.go`** — the full server, showing every route rewritten for protobuf:

```go
package main

import (
	"io"
	"log"
	"net/http"
	"strconv"
	"strings"

	"google.golang.org/protobuf/proto"
	"my-rest-api/pb" // generated package
)

var users = []*pb.User{
	{Id: 1, Name: "Alex", Email: "alex@example.com"},
	{Id: 2, Name: "Sam", Email: "sam@example.com"},
}

// Small helper so every handler writes protobuf errors consistently
func writeProtoError(w http.ResponseWriter, status int, message string) {
	w.Header().Set("Content-Type", "application/x-protobuf")
	w.WriteHeader(status)
	errBytes, _ := proto.Marshal(&pb.ErrorResponse{Error: message})
	w.Write(errBytes)
}

// GET /users
func getUsersHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/x-protobuf")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	list := &pb.UserList{Users: users}
	respBytes, err := proto.Marshal(list) // instead of json.NewEncoder(w).Encode(list)
	if err != nil {
		writeProtoError(w, http.StatusInternalServerError, "failed to encode response")
		return
	}
	w.WriteHeader(http.StatusOK)
	w.Write(respBytes)
}

// POST /users
func createUserHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/x-protobuf")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	body, err := io.ReadAll(r.Body) // read raw bytes instead of streaming into json.Decoder
	if err != nil {
		writeProtoError(w, http.StatusBadRequest, "could not read body")
		return
	}

	var newUser pb.User
	if err := proto.Unmarshal(body, &newUser); err != nil { // instead of json.Unmarshal
		writeProtoError(w, http.StatusBadRequest, "invalid protobuf body")
		return
	}

	if newUser.Name == "" || newUser.Email == "" {
		writeProtoError(w, http.StatusUnprocessableEntity, "name and email required")
		return
	}

	newUser.Id = int32(len(users) + 1)
	users = append(users, &newUser)

	respBytes, _ := proto.Marshal(&newUser)
	w.WriteHeader(http.StatusCreated)
	w.Write(respBytes)
}

// GET /users/3
func getUserByIDHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/x-protobuf")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	idStr := strings.TrimPrefix(r.URL.Path, "/users/")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		writeProtoError(w, http.StatusBadRequest, "invalid id")
		return
	}

	for _, u := range users {
		if u.Id == int32(id) {
			respBytes, _ := proto.Marshal(u)
			w.WriteHeader(http.StatusOK)
			w.Write(respBytes)
			return
		}
	}
	writeProtoError(w, http.StatusNotFound, "user not found")
}

func main() {
	http.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		switch r.Method {
		case http.MethodGet:
			getUsersHandler(w, r)
		case http.MethodPost:
			createUserHandler(w, r)
		default:
			writeProtoError(w, http.StatusMethodNotAllowed, "method not allowed")
		}
	})
	http.HandleFunc("/users/", getUserByIDHandler)

	log.Println("Server running on http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### What's actually different, line by line, from the plain JSON version

| In the JSON version | In the protobuf-over-REST version | Why |
|---|---|---|
| `type User struct { ... json:"..." }` hand-written in `main.go` | `pb.User` generated from `proto/user.proto` | Schema now lives in one place both client and server generate from |
| `json.NewDecoder(r.Body).Decode(&newUser)` | `io.ReadAll(r.Body)` then `proto.Unmarshal(body, &newUser)` | Protobuf doesn't have a streaming decoder built into `net/http` the way `encoding/json` does — you read the full byte slice first, then unmarshal it |
| `json.NewEncoder(w).Encode(newUser)` | `proto.Marshal(&newUser)` then `w.Write(respBytes)` | Same reasoning in reverse — marshal to bytes first, then write |
| `w.Header().Set("Content-Type", "application/json")` | `w.Header().Set("Content-Type", "application/x-protobuf")` | Tells the client which decoder to use on its end |
| `map[string]string{"error": "..."}` encoded as JSON | `pb.ErrorResponse{Error: "..."}` encoded as protobuf | Even your error responses need a defined message type now — you can't just throw an arbitrary map at `proto.Marshal`, protobuf requires a known message schema for everything you serialize |
| Routing (`switch r.Method`, path parsing) | **Unchanged** | This hybrid approach keeps REST's routing model entirely — only the body encoding changed |
| Status codes (200/201/400/404/422) | **Unchanged** | Still plain HTTP status codes — gRPC's separate status code system doesn't come into play here |

The biggest practical adjustment for a backend developer: **you can no longer construct a quick inline JSON object** for a response (like `map[string]string{"error": "..."}`) — every single thing you send over the wire, including errors, must be a message type defined in a `.proto` file and generated ahead of time. This is a real workflow change even though the HTTP mechanics around it don't move.

### Why you might still prefer full gRPC instead

- You lose gRPC's built-in streaming (client/server/bidirectional) — this hybrid is strictly request/response, same as REST.
- You lose the generated *service* layer (RPC method stubs) — you're still hand-writing routing and hand-calling `proto.Marshal`/`Unmarshal` yourself, rather than getting a fully generated client and server.
- You don't get gRPC's built-in status code model, deadline/timeout propagation, or interceptor ecosystem (widely used for cross-cutting concerns like logging, auth, and retries in full gRPC systems).

---

## Summary

- **Full gRPC** changes the transport (HTTP/2), the message format (protobuf), the contract (`.proto`-generated code), and requires a grpc-web + proxy layer for any browser frontend — a genuinely new architecture, not a drop-in replacement.
- **Browsers cannot call real gRPC directly** — HTTP/2 trailers aren't exposed to JavaScript, so you need grpc-web and a translating proxy (typically Envoy) in front of your gRPC backend.
- **If you just want protobuf's efficiency without gRPC's architecture**, use protobuf purely as a body serialization format over your existing REST setup — everything else (URLs, verbs, status codes, `fetch()`, load balancer, no proxy needed) stays the same, and only the marshal/unmarshal logic on both ends changes.
