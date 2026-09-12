# REST APIs Explained — A Note for Understanding the Go + React Example

This note walks through **every concept** used in the Go server + React client example, explained the way you'd explain it to a junior developer seeing REST for the first time. It's organized so you can read it top to bottom, or jump to the section you're confused about.

---

## 1. What is a REST API, really?

REST (Representational State Transfer) is just a set of **conventions** for how a client (browser, app, script) talks to a server over HTTP. It's not a special protocol — it's HTTP, used in a structured, predictable way:

- Every **resource** (a user, an order, a product) has a **URL** that represents it.
- You interact with that resource using standard **HTTP methods** (GET, POST, PUT, DELETE).
- The server responds with a **status code** (did it work?) and a **body** (the data).

That's the whole idea. Everything else (headers, auth, params) is supporting machinery around this core loop.

---

## 2. The anatomy of an HTTP request

Every request — whether from React's `fetch`, Postman, or curl — has four parts:

```
METHOD  URL
Headers: key-value pairs
Body: (optional) the actual data being sent
```

Example:
```
POST /users HTTP/1.1
Host: localhost:8080
Content-Type: application/json
Authorization: Bearer secret123

{"name": "Alex", "email": "alex@example.com"}
```

Let's break down each piece.

---

## 3. HTTP Methods (verbs)

Methods tell the server **what kind of action** you want to perform on a resource.

| Method | Meaning | Example in our code |
|---|---|---|
| **GET** | Read/fetch data. Never changes anything on the server. | `GET /users` — fetch all users |
| **POST** | Create something new. | `POST /users` — add a new user |
| **PUT** | Replace an entire existing resource. | `PUT /users/3` — overwrite user 3 completely |
| **PATCH** | Update part of a resource. | `PATCH /users/3` — just change the email |
| **DELETE** | Remove a resource. | `DELETE /users/3` — delete user 3 |
| **OPTIONS** | Ask the server "what are you allowed to do?" — used automatically by browsers for CORS (explained later). | Handled explicitly in our Go code |

**Important property: idempotency.** GET, PUT, and DELETE are supposed to be *idempotent* — calling them multiple times with the same input has the same effect as calling them once (deleting user 3 twice still ends with user 3 gone). POST is *not* idempotent — calling it twice creates two resources. This matters when you're deciding what method to use for an action, and it's why retry logic behaves differently for GET vs POST.

In our Go code, the `switch r.Method` block is exactly this — deciding what logic to run based on which verb was used.

### A separate worked example for every method

Below is one self-contained example per method — Go handler, matching React call, and what's actually happening — so you can see each verb in isolation instead of mixed together in one big handler.

Shared setup used by every example:
```go
type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

var users = []User{
	{ID: 1, Name: "Alex", Email: "alex@example.com"},
	{ID: 2, Name: "Sam", Email: "sam@example.com"},
}
```
```js
const API_BASE = "http://localhost:8080";
```

#### GET — read data, never modifies anything

```go
func getUsersHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	limitParam := r.URL.Query().Get("limit") // query param
	result := users
	if limitParam != "" {
		if limit, err := strconv.Atoi(limitParam); err == nil && limit < len(users) {
			result = users[:limit]
		}
	}

	w.WriteHeader(http.StatusOK) // 200
	json.NewEncoder(w).Encode(result)
}
```
```js
async function getUsers() {
  const res = await fetch(`${API_BASE}/users?limit=1`, {
    method: "GET",
    headers: { "Accept": "application/json" },
  });
  const data = await res.json();
  console.log(res.status, data); // 200 [{ id: 1, name: "Alex", ... }]
  return data;
}
```
**What's happening:** no request body — GET only asks for data, it never sends any. The `limit` filter is passed as a query param in the URL. `Accept` tells the server what format we want back; `Content-Type` on the response tells us how to parse what we got. `200` means "here's your data, nothing changed."

#### POST — create a new resource

```go
func createUserHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	var newUser User
	if err := json.NewDecoder(r.Body).Decode(&newUser); err != nil {
		w.WriteHeader(http.StatusBadRequest) // 400
		json.NewEncoder(w).Encode(map[string]string{"error": "invalid JSON body"})
		return
	}
	if newUser.Name == "" || newUser.Email == "" {
		w.WriteHeader(http.StatusUnprocessableEntity) // 422
		json.NewEncoder(w).Encode(map[string]string{"error": "name and email required"})
		return
	}

	newUser.ID = len(users) + 1
	users = append(users, newUser)
	w.WriteHeader(http.StatusCreated) // 201
	json.NewEncoder(w).Encode(newUser)
}
```
```js
async function createUser(name, email) {
  const res = await fetch(`${API_BASE}/users`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": "Bearer secret123",
    },
    body: JSON.stringify({ name, email }),
  });
  const data = await res.json();
  console.log(res.status, data); // 201 { id: 3, name: "...", email: "..." }
  return data;
}
```
**What's happening:** the request body carries the actual new data, serialized via `JSON.stringify`. `Content-Type: application/json` tells Go's decoder how to read that body. `201 Created` (not just 200) specifically signals "a new resource now exists." `400` means the JSON itself was broken; `422` means it parsed fine but failed validation (missing fields).

#### PUT — replace an entire existing resource

```go
func replaceUserHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	idStr := strings.TrimPrefix(r.URL.Path, "/users/") // path param
	id, err := strconv.Atoi(idStr)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	var updated User
	if err := json.NewDecoder(r.Body).Decode(&updated); err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	for i, u := range users {
		if u.ID == id {
			updated.ID = id
			users[i] = updated // full replacement
			w.WriteHeader(http.StatusOK) // 200
			json.NewEncoder(w).Encode(updated)
			return
		}
	}
	w.WriteHeader(http.StatusNotFound) // 404
}
```
```js
async function replaceUser(id, name, email) {
  const res = await fetch(`${API_BASE}/users/${id}`, {
    method: "PUT",
    headers: {
      "Content-Type": "application/json",
      "Authorization": "Bearer secret123",
    },
    body: JSON.stringify({ name, email }), // every field must be sent
  });
  const data = await res.json();
  console.log(res.status, data); // 200 { id: 2, name: "...", email: "..." }
  return data;
}
```
**What's happening:** the path param (`/users/2`) identifies *which* resource to overwrite. The entire object must be sent — if you left out `email`, a true PUT would wipe it out, because PUT means "this is the complete new version," not "merge this in." `200` on success since nothing new was *created*; `404` if the ID doesn't exist.

#### PATCH — partially update a resource

```go
func patchUserHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Access-Control-Allow-Origin", "*")

	idStr := strings.TrimPrefix(r.URL.Path, "/users/")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	var patch map[string]string // only holds fields that were actually sent
	if err := json.NewDecoder(r.Body).Decode(&patch); err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	for i, u := range users {
		if u.ID == id {
			if name, ok := patch["name"]; ok {
				users[i].Name = name
			}
			if email, ok := patch["email"]; ok {
				users[i].Email = email
			}
			w.WriteHeader(http.StatusOK) // 200
			json.NewEncoder(w).Encode(users[i])
			return
		}
	}
	w.WriteHeader(http.StatusNotFound) // 404
}
```
```js
async function updateEmail(id, newEmail) {
  const res = await fetch(`${API_BASE}/users/${id}`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json",
      "Authorization": "Bearer secret123",
    },
    body: JSON.stringify({ email: newEmail }), // only the one field we're changing
  });
  const data = await res.json();
  console.log(res.status, data); // 200 { id: 2, name: "Sam" (unchanged), email: "new@..." }
  return data;
}
```
**What's happening:** the body only contains `email` — `name` is left out on purpose and stays unchanged. Go decodes into a `map[string]string` instead of the `User` struct specifically so it can tell which keys were actually present, and only touches those. This is the core difference from PUT: PUT replaces the whole object, PATCH applies specific changes.

#### DELETE — remove a resource

```go
func deleteUserHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "*")

	idStr := strings.TrimPrefix(r.URL.Path, "/users/")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	for i, u := range users {
		if u.ID == id {
			users = append(users[:i], users[i+1:]...)
			w.WriteHeader(http.StatusNoContent) // 204 — success, nothing to return
			return
		}
	}
	w.WriteHeader(http.StatusNotFound) // 404
}
```
```js
async function deleteUser(id) {
  const res = await fetch(`${API_BASE}/users/${id}`, {
    method: "DELETE",
    headers: { "Authorization": "Bearer secret123" }, // no body needed
  });
  console.log(res.status); // 204 — no body to parse
  return res.status === 204;
}
```
**What's happening:** no request body — the path param alone identifies what to delete. No response body either: `204 No Content` means "it worked, nothing meaningful to send back." Calling `res.json()` on a 204 would actually throw, since there's nothing to parse — that's why the client only checks `res.status`.

#### OPTIONS — the automatic CORS preflight (you don't call this yourself)

```go
func router(w http.ResponseWriter, r *http.Request) {
	if r.Method == http.MethodOptions {
		w.Header().Set("Access-Control-Allow-Origin", "*")
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
		w.WriteHeader(http.StatusNoContent) // 204 — "yes, proceed"
		return
	}
	// ...normal method routing continues here
}
```
What the browser sends automatically, before the real PATCH request above:
```
OPTIONS /users/2 HTTP/1.1
Access-Control-Request-Method: PATCH
Access-Control-Request-Headers: content-type, authorization
```
**What's happening:** this exchange happens *before* your `fetch()` call's real request goes out, and you never write client code to trigger it — the browser does it automatically for "non-simple" requests (custom headers, methods other than GET/POST/HEAD). The server just needs to answer with the right `Access-Control-Allow-*` headers and a `204`. If it answers wrong, the browser blocks the real request entirely and your `fetch()` fails with a CORS error — even though the server never saw the real PATCH.

**Summary table:**

| Method | Body sent? | Body returned? | Typical success code | Idempotent? |
|---|---|---|---|---|
| GET | No | Yes | 200 | Yes |
| POST | Yes | Yes (created object) | 201 | No |
| PUT | Yes (entire object) | Yes (updated object) | 200 | Yes |
| PATCH | Yes (partial fields) | Yes (updated object) | 200 | Yes (in practice) |
| DELETE | No | No | 204 | Yes |
| OPTIONS | No | No | 204 | Yes (automatic, browser-driven) |

---

## 4. Headers — metadata about the request/response

Headers are **key-value pairs** that describe the request or response *without* being part of the actual data. Think of them as the envelope, not the letter inside.

### Request headers (client → server)

```js
headers: {
  "Content-Type": "application/json",
  "Authorization": "Bearer secret123",
  "Accept": "application/json",
}
```

- **`Content-Type`** — tells the server "the body I'm sending you is formatted as JSON." Without this, the server might not know how to parse the body correctly. In Go, `json.NewDecoder(r.Body).Decode(...)` assumes JSON — this header is the contract that promises that's actually what's coming.
- **`Authorization`** — carries credentials proving who you are. The common format is `Bearer <token>` — "Bearer" just means "the holder of this token is authorized," followed by the actual token/JWT/API key.
- **`Accept`** — tells the server what format *you* want back (e.g., "I only understand JSON, don't send me XML"). Servers can use this for **content negotiation** — deciding the response format based on what the client says it accepts.

### Response headers (server → client)

```go
w.Header().Set("Content-Type", "application/json")
w.Header().Set("Access-Control-Allow-Origin", "*")
```

- **`Content-Type`** on the response tells the *client* how to parse what it's receiving.
- **`Access-Control-Allow-*`** headers are CORS headers (explained in section 8) — they tell the browser which other origins/domains are allowed to read this response.

**Mental model:** headers are metadata that both sides use to correctly interpret and validate the exchange — they're read by the "postal system" (browser, server), not by your business logic.

---

## 5. The Body — the actual payload

The body is the real data being transferred. It only exists on requests that send data (POST, PUT, PATCH) and on virtually all responses.

**Request body (client → server):**
```js
body: JSON.stringify({ name, email })
```
`JSON.stringify` converts a JS object into a JSON *string*, because HTTP bodies are just raw bytes/text — you can't send a live JavaScript object over the wire, only a serialized version of it.

**Reading the body on the server (Go):**
```go
var newUser User
json.NewDecoder(r.Body).Decode(&newUser)
```
This does the reverse — it reads the raw JSON string from the request and *deserializes* it back into a Go struct (`User`) so your code can work with real fields (`newUser.Name`, `newUser.Email`) instead of raw text.

**Response body (server → client):**
```go
json.NewEncoder(w).Encode(newUser)
```
Same idea in reverse — take a Go struct, serialize it to JSON, write it into the HTTP response.

**On the client, reading the response body:**
```js
const data = await res.json();
```
`res.json()` reads the raw response text and parses it back into a JS object you can use (`data.name`, `data.email`).

**Key idea:** JSON is just a text format both sides agree to use. Serialization/deserialization is the process of converting between "real data structures in memory" and "text that can travel over a network."

---

## 6. Status codes — the outcome of the request

The status code is a 3-digit number telling you, at a glance, what happened — before you even look at the body.

| Range | Category | Meaning |
|---|---|---|
| 2xx | Success | The request worked |
| 3xx | Redirection | Resource moved elsewhere |
| 4xx | Client error | You (the caller) did something wrong |
| 5xx | Server error | The server broke while handling it |

Specific codes used in our example:

| Code | Name | When we used it | Why |
|---|---|---|---|
| **200** | OK | Successful `GET /users` | Standard "it worked, here's your data" |
| **201** | Created | Successful `POST /users` | Specifically means "a new resource was created" — more precise than a plain 200 |
| **204** | No Content | CORS preflight response | Request succeeded but there's no body to send back |
| **400** | Bad Request | Malformed JSON body, invalid path param | The client sent something the server literally can't parse |
| **401** | Unauthorized | Missing/invalid `Authorization` header | "I don't know who you are" |
| **404** | Not Found | Requesting a user ID that doesn't exist | The resource simply isn't there |
| **422** | Unprocessable Entity | Valid JSON, but missing required fields (`name`, `email`) | The syntax is fine, but the *content* is invalid |
| **405** | Method Not Allowed | Using an HTTP method the route doesn't support | E.g., sending DELETE to a route that only supports GET/POST |

**Why the split between 400 and 422?** 400 means "I couldn't even understand your request" (broken JSON syntax). 422 means "I understood it fine, but the data doesn't meet the rules" (e.g., you forgot to include an email). This distinction matters in real APIs and interviews.

On the client, you check this via `res.ok` (true for any 2xx status) or `res.status` directly:
```js
if (!res.ok) {
  throw new Error(`HTTP ${res.status}`);
}
```

---

## 7. Authentication — the `Authorization` header and tokens

Our example uses a fake **Bearer token**:
```
Authorization: Bearer secret123
```

**How this works conceptually (even though our example hardcodes it):**
1. A user logs in with a username/password once.
2. The server verifies credentials and issues a **token** (often a JWT — JSON Web Token) — a signed piece of data that says "this is user X, and this token is valid until Y."
3. The client stores that token (in memory, a cookie, or secure storage) and attaches it to *every subsequent request* via the `Authorization` header.
4. The server checks the token on each request instead of asking for a password again — this is what makes REST **stateless** (see section 9).

In our Go code:
```go
func isAuthorized(r *http.Request) bool {
	token := r.Header.Get("Authorization")
	return token == "Bearer secret123"
}
```
This is a simplified stand-in for what a real system would do — decode and verify a JWT signature, check an expiry timestamp, look up an API key in a database, etc. The *pattern* (read the header, validate it, reject with 401 if invalid) is the same regardless of how sophisticated the validation gets.

**Why not send username/password on every request?** Because tokens can be scoped, expired, and revoked independently, and they avoid repeatedly transmitting the actual password.

---

## 8. CORS — why the OPTIONS method exists

CORS (Cross-Origin Resource Sharing) is a **browser security feature**, not a server or REST concept per se — but you'll hit it constantly when React (`localhost:3000`) talks to Go (`localhost:8080`), because those count as two different "origins" (different ports = different origin, even on the same machine).

By default, browsers **block** JavaScript from reading a response from a different origin, unless the server explicitly says it's allowed via headers:

```go
w.Header().Set("Access-Control-Allow-Origin", "*")
w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
```

- `Access-Control-Allow-Origin` — which origins are allowed to read the response (`*` = anyone, but in production you'd restrict this to your real frontend domain).
- `Access-Control-Allow-Methods` — which HTTP methods are permitted cross-origin.
- `Access-Control-Allow-Headers` — which custom headers (like `Authorization`) the browser is allowed to send.

**Preflight requests:** for certain requests (like ones with custom headers or non-simple methods), the browser automatically sends an `OPTIONS` request *first*, asking "hey, are you okay with what I'm about to do?" before sending the real request. That's why our Go code has:
```go
if r.Method == http.MethodOptions {
	w.WriteHeader(http.StatusNoContent) // 204
	return
}
```
This just tells the browser "yes, go ahead" without doing any real work — the actual GET/POST follows immediately after.

**You never trigger this manually** — it's automatic browser behavior. You only need to make sure your server responds to it correctly.

---

## 9. Path parameters vs. query parameters

Both let you pass extra information in the URL, but they mean different things conceptually:

### Path parameters — identify *which* resource
```
GET /users/3
```
`3` here identifies a *specific* resource. It's part of the URL structure itself.

In Go:
```go
idStr := strings.TrimPrefix(r.URL.Path, "/users/")
id, _ := strconv.Atoi(idStr)
```
We manually strip the prefix to extract `3` as a string, then convert it to an integer. (In a real project, you'd use a router library like `chi` or `gorilla/mux` to do this more cleanly with named parameters like `/users/{id}`.)

### Query parameters — filter, sort, or modify *how* you fetch a resource
```
GET /users?limit=2
```
Everything after `?` is a query string — optional modifiers to the request, not identifiers.

In Go:
```go
limitParam := r.URL.Query().Get("limit")
```

**Rule of thumb:** if it identifies *which* thing you want → path param. If it filters/modifies/paginates → query param. E.g., `/orders/42?include=items` — `42` is the specific order, `include=items` is an optional modifier.

---

## 10. Statelessness

REST APIs are meant to be **stateless**: the server doesn't remember anything about you between requests. Every single request must carry everything needed to understand it — including the auth token.

This is *why* we attach `Authorization: Bearer secret123` on every call, rather than logging in once and having the server "remember" you're logged in. Each request stands completely on its own. This makes REST APIs easy to scale (any server instance can handle any request, since there's no shared memory of "who's logged in") and easy to reason about.

---

## 11. Content negotiation

This is the idea that a single endpoint could return *different formats* depending on what the client asks for, via the `Accept` header:
```
Accept: application/json
```
could instead be:
```
Accept: application/xml
```

Most modern APIs just use JSON exclusively and don't bother with this, but it's why the `Accept` header exists as a formal part of HTTP — historically, APIs supported multiple response formats and used this header to decide which one to send back.

---

## 12. Putting the full request/response cycle together

Here's the full lifecycle when React calls `POST /users`:

1. **React builds the request**: method (`POST`), headers (`Content-Type`, `Authorization`), and body (`JSON.stringify({name, email})`).
2. **Browser sends a CORS preflight** (`OPTIONS`) first, since we're sending a custom header (`Authorization`) cross-origin.
3. **Go responds to the preflight** with `204` and the `Access-Control-Allow-*` headers, telling the browser "this is fine, proceed."
4. **Browser sends the real POST request** with the actual body.
5. **Go's middleware logs the request** (method, path, headers) — useful for debugging.
6. **Go checks the `Authorization` header** — if it's missing/wrong, respond `401` immediately and stop.
7. **Go reads and parses the body** into a `User` struct. If the JSON is malformed → `400`. If required fields are missing → `422`.
8. **Go processes the request** (appends the new user), sets `Content-Type: application/json` on the response, and writes status `201 Created` plus the new user as JSON in the body.
9. **React receives the response**, checks `res.ok`/`res.status`, parses the body with `res.json()`, and updates the UI state.

Every concept in this note is a piece of that cycle — headers and body carry information, methods and status codes describe intent and outcome, auth and CORS are security gatekeeping, and path/query params shape exactly what's being requested.

---

## Quick-reference cheat sheet

| Concept | Client-side | Server-side (Go) |
|---|---|---|
| Method | `fetch(url, { method: "POST" })` | `r.Method`, `switch` statement |
| Request headers | `headers: {...}` in fetch options | `r.Header.Get("...")` |
| Response headers | Read via `res.headers.get(...)` | `w.Header().Set(...)` |
| Request body | `body: JSON.stringify(obj)` | `json.NewDecoder(r.Body).Decode(&struct)` |
| Response body | `await res.json()` | `json.NewEncoder(w).Encode(obj)` |
| Status code | `res.status`, `res.ok` | `w.WriteHeader(http.StatusXXX)` |
| Path param | Interpolated into the URL string | Parsed from `r.URL.Path` |
| Query param | `?key=value` in the URL | `r.URL.Query().Get("key")` |
| Auth token | `Authorization: "Bearer ..."` header | Checked in a middleware/handler function |
| CORS | Automatic — browser sends preflight | Must explicitly set `Access-Control-Allow-*` headers |
