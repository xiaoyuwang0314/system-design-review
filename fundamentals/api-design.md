# API Design

## REST basics

**Resources are nouns. HTTP methods are the verbs.**

| Method | Meaning | Example |
| --- | --- | --- |
| `GET` | Read | `GET /users/123` |
| `POST` | Create (the server assigns the ID) | `POST /users` |
| `PUT` | Replace a resource at a known URI | `PUT /users/123` |
| `PATCH` | Partial update | `PATCH /users/123` |
| `DELETE` | Delete | `DELETE /users/123` |

Nest resources that belong to a parent: `GET /conversations/{id}/messages`.

Avoid verbs in paths: `POST /getUser`, `POST /send`, `POST /createMessage`.

### Status codes

| Code | Meaning |
| --- | --- |
| 200 OK, 201 Created, 204 No Content | Success |
| 400 Bad Request | Invalid input |
| 401 Unauthorized vs. 403 Forbidden | 401: not authenticated. 403: authenticated but not allowed |
| 404 Not Found | Resource does not exist |
| 409 Conflict | Conflicting state, for example a seat that is already booked |
| 429 Too Many Requests | Rate limited |
| 500, 503 | Server error, service temporarily unavailable |

## Idempotency

An operation is **idempotent** if doing it once or many times has the same effect.
- `GET`, `PUT`, and `DELETE` are idempotent by definition.
- **`POST` is not.** Sending the same `POST` twice creates two resources.

Why it matters: when a request times out, the client cannot tell whether the server processed it, so it retries. Usually the first request *did* reach the server and only the response was lost. Retrying `POST /payments` can charge twice, and retrying "send message" can deliver the message twice.

**Fix: an idempotency key.**
1. The client generates a UUID **once per operation** (for example, when the user taps Send) and sends it as a header: `Idempotency-Key: 7f3a...`. Every retry reuses the **same** key.
2. The server checks whether it has seen the key. If not, it processes the request and stores the key with the result. If yes, it **returns the stored result** without processing again.
3. Keys expire after a while, for example a 24-hour TTL.
4. Two requests with the same key arriving at the same time: use a unique constraint or a lock so only one runs.

Stripe's API works this way. In a chat app, a client-generated `client_message_id` can serve as the idempotency key. It also lets the client match its optimistic message bubble with the server's acknowledgment.

## Pagination

| | Offset pagination | Cursor (keyset) pagination |
| --- | --- | --- |
| Request | `?limit=20&offset=40` | `?limit=20&cursor=abc123` |
| Query behind it | `LIMIT 20 OFFSET 40` | `WHERE id < :last_id ORDER BY id DESC LIMIT 20` |
| Pros | Simple, can jump to page N | Uses an index, so deep pages stay fast. Stable when new rows arrive |
| Cons | Slow at large offsets (the database reads and discards every skipped row). **Duplicates or skips when rows are inserted or deleted** | Cannot jump to page N |
| Good for | Admin pages, search results | **Feeds, chat history, infinite scroll** |

### Why offset breaks in chat history

History is sorted newest first, and the offset counts from the newest message. The user has loaded 40 messages and asks for `offset=40`. Meanwhile 5 new messages arrive at the top. Everything shifts down by 5, so the next page starts 5 messages too early, and the user sees 5 messages again. Deletes cause the opposite problem: messages get skipped.

A cursor says "messages older than message X". New messages at the top do not change what is older than X, so pages stay correct.

The cursor is usually an **opaque token** that encodes the last seen ID or timestamp. Clients just pass it back.

```json
{ "data": [ ... ], "next_cursor": "abc123" }
```

## REST vs. gRPC vs. GraphQL

| | REST | gRPC | GraphQL |
| --- | --- | --- | --- |
| Format | JSON over HTTP | Protobuf (binary) over HTTP/2 | One endpoint, the client's query names the fields it wants |
| Pros | Universal, cacheable, familiar | Fast, strongly typed contracts, streaming | One request returns exactly the needed data |
| Cons | Several requests to assemble data, or extra unused fields | Weak browser support (needs grpc-web) | Hard to cache, clients can write expensive queries |
| Good for | **Public and client-facing APIs** | **Internal service-to-service calls** | Many different frontends combining many resources |

Interview default: **REST for client-facing APIs, gRPC for internal calls.** L1 is enough for gRPC and GraphQL: know what they are and when to choose them.

The principles carry over across protocols:

| Principle | REST | gRPC | GraphQL |
| --- | --- | --- | --- |
| Cursor pagination | `?limit=20&cursor=abc` | `page_size` and `page_token` in the request, `next_page_token` in the response | `messages(first: 20, after: "abc")` returning `pageInfo { endCursor, hasNextPage }` |
| Idempotency | `Idempotency-Key` header | A `request_id` field in the request message | An idempotency key argument on the mutation |

## Other essentials

- **Versioning**: `/v1/...` in the path, or a header. Adding fields is backward compatible. Never remove or rename existing fields.
- **Auth**: send the token in the `Authorization` header, never in the URL, since URLs end up in logs. The server takes the user's identity from the token, not from the request body.
- **In interviews**: list three to five core endpoints with method, path, key parameters, and response shape. Do not list every endpoint.

## Worked example: chat app endpoints

**Send a message**

```
POST /v1/conversations/{conversation_id}/messages
Authorization: Bearer <token>
Idempotency-Key: <client-generated UUID>

{ "text": "hi" }
```

Response `201 Created`:

```json
{
  "message_id": "m_789",
  "conversation_id": "c_123",
  "sender_id": "u_42",
  "text": "hi",
  "created_at": "2026-10-20T18:30:00Z"
}
```

- `POST`, because the server creates a new message and assigns its ID
- The path names the resource (messages in a conversation), not an action like `/send`
- The sender comes from the auth token, so a client cannot send as someone else
- Return the created message so the client gets its ID and server timestamp

**Fetch message history**

```
GET /v1/conversations/{conversation_id}/messages?limit=50&cursor=<token>
Authorization: Bearer <token>
```

Response `200 OK`:

```json
{
  "messages": [
    { "message_id": "m_789", "sender_id": "u_42", "text": "hi", "created_at": "2026-10-20T18:30:00Z" }
  ],
  "next_cursor": "eyJiZWZvcmUiOiJtXzc0MCJ9"
}
```

- Newest first. Omit `cursor` for the latest page, pass `next_cursor` to load older messages
- `next_cursor` is `null` when there are no older messages
- The response is a list plus a cursor, not a single message

## Follow-up questions

**Why POST and not PUT for sending a message?**
`PUT` replaces a resource at a URI the client already knows, and it is idempotent. Sending a message creates a new resource whose ID the server assigns, which is what `POST` is for. (A variant is `PUT /conversations/{id}/messages/{client_message_id}`, which makes the create idempotent by design. It works, but `POST` plus an idempotency key is the common answer.)

**How do you make a retried payment safe?**
Idempotency key: the client sends the same key on every retry, and the server returns the stored result instead of charging again.

**Offset or cursor pagination for a feed?**
Cursor. New items keep arriving at the top, which shifts offsets and causes duplicates, and deep offsets get slow.

## Parked

- HATEOAS and strict REST purity
- API gateways and backend-for-frontend (BFF)
- Webhook design (retries, signatures)

## References

- Google Cloud, API Design Guide (resource-oriented design, pagination)
- Stripe API reference, "Idempotent requests"
- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 4 (Encoding and Evolution)
