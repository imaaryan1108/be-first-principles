# Session 2 — Serialization & Deserialization

## The Problem

* Client and server are often written in different languages
   * Example: Frontend in JavaScript, Backend in Rust
* Each language stores data in memory in its own way
   * A JS object and a Rust struct look completely different in memory
* So how can a Rust server understand data sent by a JS frontend, run business logic on it, and send back something JS understands again?

## The Solution

* Both sides agree on a **common, language-agnostic format**
* **Serialization** = converting in-memory data → common format (text/bytes) to send over the network
* **Deserialization** = converting common format → in-memory data of your own language

```
JS object  --serialize-->  JSON text  --network-->  deserialize  -->  Rust struct
                                                                      (business logic)
JS object  <--deserialize--  JSON text  <--network--  serialize  <--  Rust struct
```

* Example in JS
   * `JSON.stringify({ name: "Asha", age: 25 })` → `'{"name":"Asha","age":25}'` (serialize)
   * `JSON.parse('{"name":"Asha","age":25}')` → object again (deserialize)
* In Rust, a library like `serde` does the same job

## Where this fits — OSI Model

* The OSI model explains how data travels from one machine to another in 7 layers
* Each layer does one job and hands over to the next

| # | Layer | Job | Example |
|---|---|---|---|
| 7 | Application | What the app talks in | HTTP, DNS |
| 6 | Presentation | Format, encryption, compression | JSON, TLS |
| 5 | Session | Opening/closing conversations | — |
| 4 | Transport | Reliable delivery between apps | TCP, UDP |
| 3 | Network | Routing between machines | IP |
| 2 | Data Link | Delivery on the local network | Ethernet, Wi-Fi |
| 1 | Physical | Actual bits on wire/air | Cables, radio |

* Top layer = Application, bottom layer = Physical
* On the sending side, data goes top → bottom; on the receiving side, bottom → top
* Serialization conceptually belongs to the upper layers (Application / Presentation)
* Key idea: at every layer, both sides must follow the same set of rules — serialization standards are those rules for the **data itself**

## Types of Serialization Standards

* **Text-based** (human-readable)
   * **JSON**
      * `{"name": "Asha", "age": 25, "skills": ["js", "rust"]}`
      * Most popular for web APIs
   * **XML**
      * `<user><name>Asha</name><age>25</age></user>`
      * Older, verbose, still used in banking / enterprise / SOAP APIs
   * **YAML**
      * Indentation-based, very readable
      * Mostly used for config files (Docker Compose, Kubernetes, CI pipelines), not for APIs
* **Binary** (not human-readable, but smaller and faster)
   * **Protocol Buffers (Protobuf)** — used with gRPC, common between internal services
   * **MessagePack, Avro** — similar idea
   * Trade-off: faster and smaller, but you can't just read it by eye and both sides need a shared schema

## JSON (most popular)

* Why JSON won
   * Native to JavaScript, and every language has a JSON library
   * Easy to read and write
   * Lighter than XML
* Data types it supports
   * String → `"hello"`
   * Number → `42`, `3.14`
   * Boolean → `true` / `false`
   * Null → `null`
   * Array → `[1, 2, 3]`
   * Object → `{"key": "value"}`
* Rules to remember
   * Keys must be in double quotes
   * No comments allowed
   * No trailing commas
   * No native date type → dates are sent as strings, usually ISO format `"2026-09-26T10:00:00Z"`
* Sent with header `Content-Type: application/json`

## Good Practices

* Always **validate** incoming data after deserializing
   * Never trust the client — check types, required fields, lengths
   * A request `{"age": "abc"}` should be rejected with `400 Bad Request`
* Don't serialize everything blindly
   * e.g. never send a user's password hash back in the response
* Keep field naming consistent across APIs (`camelCase` or `snake_case`, pick one)
