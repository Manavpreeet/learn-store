# REST vs GraphQL vs gRPC: A Deep Technical Dive

*Understanding API protocols from bytes to production systems*

---

## Part I: Foundation - Understanding the Wire

Before we dive into protocols, we need to understand what actually travels across the network.

### The Fundamentals: Bytes and Characters

Everything sent over a network is **just bytes**. When you see text, numbers, or symbols in your API responses, they're all encoded as bytes:

- ASCII characters = 1 byte each
- A byte = 8 bits
- "Hello" = 5 bytes on the wire

This matters because when we talk about protocol efficiency, we're really talking about how many bytes are sent for each request and response.

### CRLF: The Hidden Foundation of HTTP

**CRLF** (Carriage Return + Line Feed) is one of those details that seems trivial until you realize it's fundamental to how HTTP works.

```
CRLF = \r\n = 2 bytes
```

| Symbol | ASCII Code | Purpose |
|--------|-----------|---------|
| `\r` | 13 | Carriage Return (move to start of line) |
| `\n` | 10 | Line Feed (move to next line) |

This convention came from early computers and typewriters. HTTP/1.1 uses CRLF everywhere:

- Every header line ends with CRLF
- Headers section ends with an empty line (CRLF + CRLF)
- The body starts immediately after that empty line

**Why this matters:** Malformed CRLF breaks servers. Missing the blank line between headers and body causes parsing failures. This seemingly minor detail is critical to understanding why HTTP/1.1 has the performance characteristics it does.

---

## Part II: REST - The HTTP Protocol

### Raw HTTP Request Structure

Here's what an actual HTTP/1.1 request looks like at the byte level:

```text
POST /users HTTP/1.1\r\n
Host: api.example.com\r\n
User-Agent: my-client/1.0\r\n
Accept: application/json\r\n
Content-Type: application/json\r\n
Content-Length: 27\r\n
\r\n
{"name":"Manav","id":1}
```

Let's count the bytes in one header:

```
Host: api.example.com\r\n
```

- `Host:` → 5 bytes
- space → 1 byte
- `api.example.com` → 15 bytes
- `\r\n` → 2 bytes
- **Total: 23 bytes**

Now multiply that by 10-20 headers per request. You're sending hundreds of bytes of metadata for every single request, even if you're just fetching a tiny piece of data.

### How Servers Parse HTTP/1.1

Conceptually, servers parse HTTP/1.1 like this:

```c
while (true) {
  line = readUntilCRLF(socket);
  if (line.length == 0) break;   // Empty line means headers end
  parseHeader(line);              // Parse "key: value"
}
// Body starts here
```

This text-based parsing is:
- Simple to implement
- Easy to debug
- Human-readable
- **Computationally expensive**

### The Performance Problem

HTTP/1.1 has fundamental inefficiencies at scale:

1. **Header repetition**: Same headers sent with every request
2. **Text parsing overhead**: Servers must parse text on every request
3. **Connection limitations**: Need multiple TCP connections for parallel requests
4. **Head-of-line blocking**: One slow request blocks others on the same connection

These aren't bugs—they're inherent to the protocol design.

### REST: Modern Implementation

**Server (Node.js/Express - 2024):**

```javascript
import express from 'express';
import { body, validationResult } from 'express-validator';

const app = express();
app.use(express.json());

// Modern REST endpoint with validation
app.post('/api/users',
  body('name').isString().trim().isLength({ min: 1, max: 100 }),
  body('email').isEmail().normalizeEmail(),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    try {
      const user = await db.users.create({
        name: req.body.name,
        email: req.body.email,
      });
      
      res.status(201).json({
        data: user,
        links: {
          self: `/api/users/${user.id}`,
          orders: `/api/users/${user.id}/orders`
        }
      });
    } catch (error) {
      res.status(500).json({ error: 'Internal server error' });
    }
  }
);

// Nested resource endpoint
app.get('/api/users/:id/orders', async (req, res) => {
  const orders = await db.orders.findAll({
    where: { userId: req.params.id },
    include: ['items', 'payments']
  });
  
  res.json({
    data: orders,
    meta: { count: orders.length }
  });
});
```

**Client (Modern Fetch API):**

```javascript
// Modern REST client with error handling
class UserClient {
  constructor(baseURL) {
    this.baseURL = baseURL;
  }

  async createUser(userData) {
    const response = await fetch(`${this.baseURL}/api/users`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.token}`
      },
      body: JSON.stringify(userData)
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return response.json();
  }

  async getUserOrders(userId) {
    const response = await fetch(
      `${this.baseURL}/api/users/${userId}/orders`
    );
    return response.json();
  }
}
```


---

## Part III: HTTP/2 - The Binary Evolution

HTTP/2 keeps HTTP semantics (methods, headers, status codes) but completely changes how data travels on the wire.

### The Mental Model: Multiplexed Streams

```
Client
  │
  └─── One TCP Connection
         │
         └─── HTTP/2 Connection
                ├─── Stream 1 (GET /users)
                ├─── Stream 3 (POST /orders)
                ├─── Stream 5 (GET /products)
                └─── Stream 7 (GET /images)
```

Key insight: **Multiple requests share one connection**, each identified by a stream ID.

### Binary Frames

HTTP/2 splits everything into **frames**. Each frame has a 9-byte header:

| Field | Size | Purpose |
|-------|------|---------|
| Length | 3 bytes | Payload size |
| Type | 1 byte | HEADERS, DATA, SETTINGS, etc. |
| Flags | 1 byte | END_STREAM, END_HEADERS, etc. |
| Stream ID | 4 bytes | Which request/response this belongs to |

**Frame types you'll encounter:**

- **HEADERS**: Method, path, headers (compressed)
- **DATA**: Request/response body
- **SETTINGS**: Connection configuration
- **WINDOW_UPDATE**: Flow control signals

### HPACK: Header Compression Deep Dive

HPACK is where HTTP/2 gets a massive efficiency boost.

**The Problem:** HTTP/1.1 sends the same headers repeatedly:
```
Authorization: Bearer eyJhbGc...  (hundreds of bytes)
User-Agent: Mozilla/5.0...
Accept: application/json
```

**The Solution:** HPACK uses two tables:

#### Static Table (Built-in)

HTTP/2 defines a static table of common headers:

| Index | Header |
|-------|--------|
| 1 | `:authority` |
| 2 | `:method: GET` |
| 3 | `:method: POST` |
| 4 | `:path: /` |

Instead of sending `:method: GET` (14 bytes), send index `2` (1-2 bytes).

#### Dynamic Table (Connection-specific)

The first time you send:
```
authorization: bearer abc123xyz789...
```

The server stores it in the dynamic table at index 62 (for example). Next request, you just send:
```
Index: 62
```

**Byte savings:** Repeated headers that took 400-800 bytes in HTTP/1.1 can be reduced to 20-60 bytes in HTTP/2 after the connection warms up. That's a **10-30x reduction**.

---

## Part IV: GraphQL - Flexibility Meets Complexity

### The Problem GraphQL Solves

Traditional REST scenario:
- Need user data: `GET /users/1`
- Need their orders: `GET /users/1/orders`
- Need order items: `GET /orders/123/items`

Problems:
- **Over-fetching**: Get entire user object when you only need the name
- **Under-fetching**: Multiple round trips to assemble related data
- **Network overhead**: Each request has full HTTP headers

GraphQL's promise:
> "Specify exactly what data you want in one request"

### What GraphQL Actually Is

GraphQL runs over HTTP (usually) but is fundamentally:
- A **query language** for your data
- A **schema** that defines your data graph
- An **execution engine** that resolves queries

**Important:** GraphQL is NOT automatically fast. It's not a database, and it's not a transport protocol.

### A Real GraphQL Request

```json
POST /graphql HTTP/1.1
Content-Type: application/json

{
  "query": "query { user(id: 1) { name orders { id total } } }"
}
```

HTTP is just the carrier. The real work happens in the GraphQL execution engine.

### The Execution Pipeline

```
Client Query
    ↓
HTTP Handler (/graphql)
    ↓
Parse Query → Abstract Syntax Tree (AST)
    ↓
Validate AST against Schema
    ↓
Execute Operation
    ↓
Call Resolvers (field by field)
    ↓
Fetch from Database/Services
    ↓
Build Response {data, errors}
    ↓
Send to Client
```

### GraphQL: Modern Implementation

**Schema Definition (GraphQL SDL):**

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!
  createdAt: DateTime!
}

type Order {
  id: ID!
  total: Float!
  items: [OrderItem!]!
  status: OrderStatus!
}

type OrderItem {
  id: ID!
  product: Product!
  quantity: Int!
  price: Float!
}

type Product {
  id: ID!
  name: String!
  reviews: [Review!]!
}

type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
}

type Mutation {
  createUser(name: String!, email: String!): User!
}
```

**Server (Apollo Server - Modern):**

```javascript
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';
import DataLoader from 'dataloader';

// DataLoader for batching (solves N+1)
class Loaders {
  constructor(db) {
    this.db = db;
    
    // Batch load orders by user IDs
    this.ordersByUserId = new DataLoader(async (userIds) => {
      const orders = await db.orders.findAll({
        where: { userId: userIds }
      });
      
      // Group orders by userId
      const orderMap = new Map();
      userIds.forEach(id => orderMap.set(id, []));
      orders.forEach(order => {
        orderMap.get(order.userId).push(order);
      });
      
      return userIds.map(id => orderMap.get(id));
    });
    
    // Batch load products
    this.products = new DataLoader(async (productIds) => {
      const products = await db.products.findAll({
        where: { id: productIds }
      });
      
      const productMap = new Map();
      products.forEach(p => productMap.set(p.id, p));
      
      return productIds.map(id => productMap.get(id));
    });
  }
}

const resolvers = {
  Query: {
    user: async (parent, { id }, { db }) => {
      return db.users.findByPk(id);
    },
    
    users: async (parent, { limit = 10, offset = 0 }, { db }) => {
      return db.users.findAll({ limit, offset });
    }
  },
  
  User: {
    // Using DataLoader to avoid N+1
    orders: async (parent, args, { loaders }) => {
      return loaders.ordersByUserId.load(parent.id);
    }
  },
  
  Order: {
    items: async (parent, args, { db }) => {
      return db.orderItems.findAll({
        where: { orderId: parent.id }
      });
    }
  },
  
  OrderItem: {
    // Batched product loading
    product: async (parent, args, { loaders }) => {
      return loaders.products.load(parent.productId);
    }
  },
  
  Mutation: {
    createUser: async (parent, { name, email }, { db }) => {
      return db.users.create({ name, email });
    }
  }
};

const server = new ApolloServer({
  typeDefs,
  resolvers,
  // Query depth limiting
  validationRules: [depthLimit(5)],
});

const { url } = await startStandaloneServer(server, {
  context: async ({ req }) => ({
    db,
    loaders: new Loaders(db),
    user: await authenticateUser(req)
  }),
});
```

**Client (Modern Apollo Client):**

```javascript
import { ApolloClient, InMemoryCache, gql } from '@apollo/client';

const client = new ApolloClient({
  uri: 'http://localhost:4000/graphql',
  cache: new InMemoryCache()
});

// Query with fragments for reusability
const GET_USER_WITH_ORDERS = gql`
  query GetUserWithOrders($userId: ID!) {
    user(id: $userId) {
      ...UserFields
      orders {
        ...OrderFields
        items {
          ...OrderItemFields
        }
      }
    }
  }
  
  fragment UserFields on User {
    id
    name
    email
  }
  
  fragment OrderFields on Order {
    id
    total
    status
  }
  
  fragment OrderItemFields on OrderItem {
    id
    quantity
    price
    product {
      id
      name
    }
  }
`;

async function fetchUserData(userId) {
  const { data, loading, error } = await client.query({
    query: GET_USER_WITH_ORDERS,
    variables: { userId }
  });
  
  return data.user;
}
```

### GraphQL: Legacy Implementation (Pre-DataLoader Era)

**Server (Express + GraphQL - Naive Implementation):**

```javascript
const express = require('express');
const { graphqlHTTP } = require('express-graphql');
const { buildSchema } = require('graphql');

// Schema defined as string (old style)
const schema = buildSchema(`
  type User {
    id: ID!
    name: String!
    orders: [Order!]!
  }
  
  type Order {
    id: ID!
    total: Float!
  }
  
  type Query {
    user(id: ID!): User
  }
`);

// Naive resolvers - N+1 problem everywhere!
const root = {
  user: ({ id }) => {
    // First query
    const user = db.query('SELECT * FROM users WHERE id = ?', [id])[0];
    
    // N+1 Problem: This gets called for EVERY user
    user.orders = db.query(
      'SELECT * FROM orders WHERE user_id = ?',
      [user.id]
    );
    
    return user;
  }
};

const app = express();

// No batching, no caching, no depth limiting
app.use('/graphql', graphqlHTTP({
  schema: schema,
  rootValue: root,
  graphiql: true, // Development UI
}));

app.listen(4000);
```

**The N+1 Problem Visualized:**

```javascript
// This query...
const query = `
  {
    users {
      id
      name
      orders {
        id
        total
      }
    }
  }
`;

// ...generates these database queries:
// 1. SELECT * FROM users          (1 query)
// 2. SELECT * FROM orders WHERE user_id = 1   (query per user)
// 3. SELECT * FROM orders WHERE user_id = 2
// 4. SELECT * FROM orders WHERE user_id = 3
// ... 
// N. SELECT * FROM orders WHERE user_id = N

// Total: 1 + N queries!
// For 1000 users = 1001 database queries
```

### Production GraphQL Query Cost Analysis

**Implementing Query Cost Analysis:**

```javascript
import { GraphQLError } from 'graphql';

class QueryComplexity {
  constructor(maxCost = 1000) {
    this.maxCost = maxCost;
  }

  // Assign costs to field types
  getFieldCost(fieldName, fieldType, args) {
    const costs = {
      // Simple scalar fields
      'id': 0,
      'name': 1,
      'email': 1,
      
      // Relationships (more expensive)
      'orders': 10,
      'reviews': 15,
      'items': 10,
      
      // Expensive operations
      'search': 50,
      'recommendations': 100
    };
    
    let baseCost = costs[fieldName] || 5;
    
    // Multiply by limit/first argument
    if (args.limit) baseCost *= args.limit;
    if (args.first) baseCost *= args.first;
    
    return baseCost;
  }

  calculateCost(document, variableValues) {
    let totalCost = 0;
    
    // Walk the AST
    visit(document, {
      Field(node) {
        const fieldName = node.name.value;
        const args = getArgumentValues(node, variableValues);
        
        const cost = this.getFieldCost(fieldName, node.type, args);
        totalCost += cost;
        
        // Nested fields multiply cost
        if (node.selectionSet) {
          totalCost *= 1.5;
        }
      }
    });
    
    if (totalCost > this.maxCost) {
      throw new GraphQLError(
        `Query cost ${totalCost} exceeds maximum ${this.maxCost}`
      );
    }
    
    return totalCost;
  }
}

// Usage in Apollo Server
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    {
      requestDidStart() {
        const complexity = new QueryComplexity(1000);
        
        return {
          didResolveOperation({ document, request }) {
            const cost = complexity.calculateCost(
              document,
              request.variables
            );
            
            console.log(`Query cost: ${cost}`);
          }
        };
      }
    }
  ]
});
```

**Example Query Costs:**

```graphql
# Low cost query (~15 points)
{
  user(id: 1) {
    id        # 0
    name      # 1
    email     # 1
  }
}

# Medium cost query (~150 points)
{
  users(limit: 10) {           # 10x multiplier
    id
    name
    orders {                    # 10 * 10 = 100
      id
      total
    }
  }
}

# High cost query (~5000 points) - REJECTED
{
  users(limit: 100) {           # 100x multiplier
    orders(limit: 50) {         # 50x multiplier per user
      items {                   # 10 per order
        product {
          reviews(limit: 20) {  # Exponential explosion!
            author {
              posts {
                comments {
                  # This is why depth limits exist
                }
              }
            }
          }
        }
      }
    }
  }
}
```

---

## Part V: gRPC - The RPC Renaissance

### What RPC Actually Means

**RPC** = Remote Procedure Call

The idea: Call a function on another machine as if it were local.

**Instead of:**
```http
POST /api/users
Content-Type: application/json

{"name": "John"}
```

**You write:**
```javascript
const user = await client.createUser({ name: "John" })
```

RPC hides:
- URLs and endpoints
- HTTP verbs
- JSON serialization
- Error handling boilerplate

### The Stub: Generated Code That Does the Work

A **stub** is auto-generated code that handles all the networking.

**Client stub:**
- Looks like a normal function
- Serializes your request to bytes
- Sends over network
- Deserializes response

**Server stub:**
- Receives bytes from network
- Deserializes to objects
- Calls your actual function
- Serializes result back

You define your API once in a `.proto` file, and the stub generator creates all this code for you.

### Protocol Buffers Encoding Format

**What is Protobuf?**

Protocol Buffers is a binary serialization format. Unlike JSON (text), Protobuf is:
- Compact (smaller size)
- Fast to parse
- Strongly typed
- Schema-driven

**How Protobuf Encodes Data:**

Each field in a Protobuf message is encoded as:
```
[field_number | wire_type] [value]
```

**Wire Types:**

| Wire Type | Encoding | Used For |
|-----------|----------|----------|
| 0 | Varint | int32, int64, bool |
| 1 | 64-bit | double, fixed64 |
| 2 | Length-delimited | string, bytes, nested messages |
| 5 | 32-bit | float, fixed32 |

**Example Encoding:**

```protobuf
message User {
  int32 id = 1;
  string name = 2;
}
```

Message: `{ id: 150, name: "John" }`

**Binary encoding (hex):**
```
08 96 01        // Field 1 (id): 150 encoded as varint
12 04           // Field 2 (name): length-delimited, 4 bytes
4a 6f 68 6e     // "John" in ASCII
```

**Breakdown:**
- `08` = field number 1, wire type 0 (varint)
- `96 01` = 150 as varint (uses fewer bytes than int32)
- `12` = field number 2, wire type 2 (length-delimited)
- `04` = length of string (4 bytes)
- `4a 6f 68 6e` = "John"

**Total: 9 bytes**

**Same data in JSON:**
```json
{"id":150,"name":"John"}
```
**Total: 24 bytes**

**Protobuf saved 60% of the bandwidth!**

**Varint Encoding (Important Detail):**

Protobuf uses variable-length integers:
- Small numbers use fewer bytes
- Each byte has a continuation bit

```
Number   Varint bytes
1        0x01           (1 byte)
127      0x7F           (1 byte)
128      0x80 0x01      (2 bytes)
150      0x96 0x01      (2 bytes)
16384    0x80 0x80 0x01 (3 bytes)
```

This is why Protobuf is so efficient for typical data.

### gRPC: Modern Implementation

**Proto Definition:**

```protobuf
syntax = "proto3";

package user.v1;

// User service definition
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  
  // Server streaming
  rpc ListUserOrders(ListOrdersRequest) returns (stream Order);
  
  // Client streaming
  rpc CreateBatchUsers(stream CreateUserRequest) returns (BatchResponse);
  
  // Bidirectional streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  google.protobuf.Timestamp created_at = 4;
}

message Order {
  int64 id = 1;
  int64 user_id = 2;
  double total = 3;
  OrderStatus status = 4;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_COMPLETED = 2;
  ORDER_STATUS_CANCELLED = 3;
}

message ListOrdersRequest {
  int64 user_id = 1;
  int32 limit = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message BatchResponse {
  int32 created_count = 1;
  repeated int64 user_ids = 2;
}

message ChatMessage {
  int64 user_id = 1;
  string text = 2;
  google.protobuf.Timestamp timestamp = 3;
}
```

**Server Implementation (Node.js):**

```javascript
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

const packageDefinition = protoLoader.loadSync('user.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const proto = grpc.loadPackageDefinition(packageDefinition).user.v1;

// Implement service methods
const userService = {
  // Unary RPC
  GetUser: async (call, callback) => {
    const userId = call.request.id;
    
    try {
      const user = await db.users.findByPk(userId);
      
      if (!user) {
        return callback({
          code: grpc.status.NOT_FOUND,
          details: `User ${userId} not found`
        });
      }
      
      callback(null, { user });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        details: error.message
      });
    }
  },
  
  // Server streaming RPC
  ListUserOrders: async (call) => {
    const userId = call.request.user_id;
    const limit = call.request.limit || 100;
    
    try {
      const orders = await db.orders.findAll({
        where: { userId },
        limit
      });
      
      // Stream each order to client
      for (const order of orders) {
        call.write({
          id: order.id,
          user_id: order.userId,
          total: order.total,
          status: order.status
        });
      }
      
      call.end();
    } catch (error) {
      call.destroy(new Error(error.message));
    }
  },
  
  // Client streaming RPC
  CreateBatchUsers: (call, callback) => {
    const createdUsers = [];
    
    call.on('data', async (request) => {
      try {
        const user = await db.users.create({
          name: request.name,
          email: request.email
        });
        createdUsers.push(user.id);
      } catch (error) {
        call.destroy(error);
      }
    });
    
    call.on('end', () => {
      callback(null, {
        created_count: createdUsers.length,
        user_ids: createdUsers
      });
    });
    
    call.on('error', (error) => {
      callback({
        code: grpc.status.INTERNAL,
        details: error.message
      });
    });
  },
  
  // Bidirectional streaming
  Chat: (call) => {
    call.on('data', (message) => {
      console.log('Received:', message);
      
      // Broadcast to all connected clients (simplified)
      call.write({
        user_id: message.user_id,
        text: message.text,
        timestamp: new Date().toISOString()
      });
    });
    
    call.on('end', () => {
      call.end();
    });
  }
};

// Create and start server
const server = new grpc.Server();

server.addService(proto.UserService.service, userService);

// Add middleware for auth, logging, etc.
server.bindAsync(
  '0.0.0.0:50051',
  grpc.ServerCredentials.createInsecure(),
  (error, port) => {
    if (error) {
      console.error('Failed to bind:', error);
      return;
    }
    console.log(`Server running on port ${port}`);
    server.start();
  }
);
```

**Client Implementation:**

```javascript
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

const packageDefinition = protoLoader.loadSync('user.proto');
const proto = grpc.loadPackageDefinition(packageDefinition).user.v1;

// Create client
const client = new proto.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Unary call with deadline
async function getUser(userId) {
  return new Promise((resolve, reject) => {
    const deadline = new Date();
    deadline.setSeconds(deadline.getSeconds() + 5); // 5 second timeout
    
    client.GetUser(
      { id: userId },
      { deadline },
      (error, response) => {
        if (error) {
          reject(error);
        } else {
          resolve(response.user);
        }
      }
    );
  });
}

// Server streaming
async function listOrders(userId) {
  const call = client.ListUserOrders({ user_id: userId, limit: 50 });
  
  call.on('data', (order) => {
    console.log('Received order:', order);
  });
  
  call.on('end', () => {
    console.log('Stream ended');
  });
  
  call.on('error', (error) => {
    console.error('Stream error:', error);
  });
}

// Client streaming
async function batchCreateUsers(users) {
  return new Promise((resolve, reject) => {
    const call = client.CreateBatchUsers((error, response) => {
      if (error) {
        reject(error);
      } else {
        resolve(response);
      }
    });
    
    users.forEach(user => {
      call.write({ name: user.name, email: user.email });
    });
    
    call.end();
  });
}
```

### gRPC: Legacy/Raw Implementation

**Before gRPC Libraries Existed - Manual Implementation:**

```javascript
// Conceptual: What gRPC does under the hood
class ManualGrpcClient {
  constructor(host, port) {
    this.http2Client = http2.connect(`https://${host}:${port}`);
  }
  
  makeUnaryCall(method, requestMessage) {
    return new Promise((resolve, reject) => {
      // Serialize protobuf manually
      const serialized = requestMessage.serializeBinary();
      
      // Create gRPC frame (5 bytes + message)
      const frame = Buffer.alloc(5 + serialized.length);
      frame[0] = 0; // Compression flag (0 = not compressed)
      frame.writeUInt32BE(serialized.length, 1); // Message length
      serialized.copy(frame, 5);
      
      // Make HTTP/2 request
      const req = this.http2Client.request({
        ':method': 'POST',
        ':path': `/user.v1.UserService/${method}`,
        ':scheme': 'https',
        'content-type': 'application/grpc+proto',
        'te': 'trailers',
        'grpc-timeout': '5S'
      });
      
      req.write(frame);
      req.end();
      
      const chunks = [];
      
      req.on('data', (chunk) => {
        chunks.push(chunk);
      });
      
      req.on('end', () => {
        const buffer = Buffer.concat(chunks);
        
        // Parse gRPC frame
        const compressed = buffer[0];
        const messageLength = buffer.readUInt32BE(1);
        const messageBytes = buffer.slice(5, 5 + messageLength);
        
        // Deserialize protobuf
        const response = ResponseMessage.deserializeBinary(messageBytes);
        resolve(response);
      });
      
      req.on('error', reject);
    });
  }
}
```

### gRPC Load Balancer Internals

**The Problem:** gRPC uses long-lived HTTP/2 connections. Traditional L4 load balancers can't balance individual RPC calls.

**Load Balancing Strategies:**

**1. Client-Side Load Balancing (Recommended):**

```javascript
import * as grpc from '@grpc/grpc-js';

// Service discovery returns multiple backend IPs
const serviceConfig = {
  loadBalancingConfig: [{
    round_robin: {} // or pick_first, grpclb
  }]
};

// Client automatically balances across endpoints
const client = new proto.UserService(
  'dns:///user-service.example.com:50051', // Resolves to multiple IPs
  grpc.credentials.createInsecure(),
  {
    'grpc.service_config': JSON.stringify(serviceConfig)
  }
);

// Each RPC call goes to a different backend
await client.GetUser({ id: 1 }); // → Backend A
await client.GetUser({ id: 2 }); // → Backend B
await client.GetUser({ id: 3 }); // → Backend C
```

**2. Proxy-Based Load Balancing (Envoy):**

```yaml
# Envoy configuration for gRPC load balancing
static_resources:
  listeners:
  - name: grpc_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 50051
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: grpc_inbound
          codec_type: AUTO
          http2_protocol_options:
            max_concurrent_streams: 100
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                  grpc: {}
                route:
                  cluster: user_service
                  timeout: 30s
  
  clusters:
  - name: user_service
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: user_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: backend1.example.com
                port_value: 50051
        - endpoint:
            address:
              socket_address:
                address: backend2.example.com
                port_value: 50051
        - endpoint:
            address:
              socket_address:
                address: backend3.example.com
                port_value: 50051
```

**3. Load Balancing with Health Checks:**

```javascript
// Server-side health check implementation
const healthService = {
  Check: (call, callback) => {
    // Check if service is healthy
    const isHealthy = checkDatabaseConnection() && 
                     checkMemoryUsage() &&
                     checkCPUUsage();
    
    if (isHealthy) {
      callback(null, { status: 'SERVING' });
    } else {
      callback(null, { status: 'NOT_SERVING' });
    }
  },
  
  Watch: (call) => {
    // Stream health status changes
    const interval = setInterval(() => {
      const isHealthy = checkServiceHealth();
      call.write({
        status: isHealthy ? 'SERVING' : 'NOT_SERVING'
      });
    }, 5000);
    
    call.on('cancelled', () => {
      clearInterval(interval);
    });
  }
};

// Register health service
server.addService(
  grpc.health.v1.Health.service,
  healthService
);
```

**Load Balancer Visualization:**

```
Client Application
       ↓
Client-Side LB (Round Robin)
       ├─────────┬─────────┬─────────┐
       ↓         ↓         ↓         ↓
   Backend 1  Backend 2  Backend 3  Backend 4
   (Healthy)  (Healthy)  (Unhealthy) (Healthy)
       ↓         ↓         ✗         ↓
   Database  Database    ---     Database
   
Health checks remove Backend 3 from rotation
Client LB distributes across 1, 2, 4 only
```

### gRPC Architecture: The Full Stack

```
Your Application Code
         ↓
Generated Client Stub
         ↓
Protocol Buffers Encoder
         ↓
gRPC Framing Layer
         ↓
HTTP/2 Stream
         ↓
TCP Connection
         ↓
Server gRPC Runtime
         ↓
Generated Server Stub
         ↓
Your Business Logic
```

### What Happens When You Call client.getUser()

Step by step:

1. You call `client.getUser({ id: 1 })`
2. Stub validates types (compile-time safe)
3. Protobuf encodes request to binary
4. gRPC adds 5-byte message frame
5. Message written into HTTP/2 DATA frame
6. Bytes travel over network
7. Server receives and parses frame
8. Server stub decodes Protobuf
9. Your actual function executes
10. Response travels back the same way

All of this happens in **microseconds** for small messages.

### Common gRPC Failure Modes

| Status Code | Likely Cause | How to Debug |
|-------------|--------------|--------------|
| UNAVAILABLE | Load balancer doesn't support HTTP/2 | Check LB config |
| DEADLINE_EXCEEDED | Server too slow or network timeout | Check server logs, increase deadline |
| INTERNAL | Protobuf version mismatch | Verify client/server proto versions |
| PERMISSION_DENIED | Missing/invalid metadata (auth) | Check auth headers |
| UNIMPLEMENTED | Method not found on server | Verify proto definitions match |

---

## Part VI: HTTP/3 and QUIC - The Future

### Why HTTP/3 Exists

HTTP/2 still has a fundamental problem: **TCP head-of-line blocking**.

**The Problem:**

```
HTTP/2 over TCP:
Stream 1: ████████░░░░ (Packet lost!)
Stream 3: ░░░░░░░░░░░░ (Waiting for Stream 1's lost packet)
Stream 5: ░░░░░░░░░░░░ (Also blocked!)

All streams blocked until TCP retransmits the lost packet
```

Even though HTTP/2 has independent streams, TCP doesn't know about them. One lost packet blocks everything.

### Enter QUIC: UDP-Based Transport

**QUIC** = Quick UDP Internet Connections

Key insight: Build reliability on top of UDP instead of using TCP.

**HTTP/3 = HTTP/2 semantics + QUIC transport**

### QUIC Architecture

```
Application Data (HTTP/3)
         ↓
QUIC Streams (multiplexed)
         ↓
QUIC Packets (with recovery)
         ↓
UDP (unreliable)
         ↓
Network
```

### QUIC Advantages

**1. No Head-of-Line Blocking:**

```
QUIC:
Stream 1: ████████✗░░░ (Packet lost - only Stream 1 affected)
Stream 3: ████████████ (Continues independently)
Stream 5: ████████████ (Also continues)
```

**2. Faster Connection Establishment:**

```
TCP + TLS 1.3:
Client → Server: SYN
Server → Client: SYN-ACK
Client → Server: ACK + ClientHello
Server → Client: ServerHello + Certificate
Client → Server: Finished
Total: 2-3 RTT before data transfer

QUIC:
Client → Server: Initial packet (includes crypto handshake)
Server → Client: Response
Total: 1 RTT before data transfer

QUIC with 0-RTT:
Client → Server: Data immediately (using cached crypto)
Total: 0 RTT!
```

**3. Connection Migration:**

```
QUIC Connection ID: abc123xyz789

Phone on WiFi:
IP: 192.168.1.5:40001 → Server
Connection ID: abc123xyz789

Phone switches to Cellular:
IP: 10.20.30.40:50002 → Server
Connection ID: abc123xyz789 (same!)

Server recognizes connection continues
No reconnection needed!
```

TCP connections break when IP changes. QUIC connections survive.

### HTTP/3 Frame Format

HTTP/3 uses similar frames to HTTP/2 but encoded differently:

```
QUIC Stream
   ↓
HTTP/3 Frame:
   [Frame Type (varint)]
   [Frame Length (varint)]
   [Frame Payload]
```

**Frame Types:**
- DATA: Same as HTTP/2
- HEADERS: Same as HTTP/2 (still uses QPACK compression)
- SETTINGS: Connection settings
- PUSH_PROMISE: Server push
- GOAWAY: Shutdown notification

### QPACK: Header Compression for HTTP/3

HTTP/3 can't use HPACK because of ordering issues. **QPACK** solves this:

```
HPACK (HTTP/2):
Stream 1: Uses dynamic table entry 62
Stream 3: Waits for Stream 1 to update table
         (head-of-line blocking at header level)

QPACK (HTTP/3):
Stream 1: Uses dynamic table entry 62
Stream 3: Can use static table or defer
         (independent of Stream 1)
```

### HTTP/3 in Production (Node.js Example)

```javascript
import { createServer } from 'http3';
import { readFileSync } from 'fs';

// HTTP/3 requires TLS
const server = createServer({
  key: readFileSync('key.pem'),
  cert: readFileSync('cert.pem'),
  alpn: ['h3'] // Application-Layer Protocol Negotiation
}, (req, res) => {
  console.log('HTTP/3 request:', req.url);
  
  res.writeHead(200, {
    'content-type': 'application/json'
  });
  
  res.end(JSON.stringify({
    protocol: 'HTTP/3',
    transport: 'QUIC',
    stream: req.stream.id
  }));
});

server.listen(443, () => {
  console.log('HTTP/3 server running on port 443');
});
```

**Client (Fetch with HTTP/3):**

```javascript
// Modern browsers automatically use HTTP/3 if available
const response = await fetch('https://example.com/api/users', {
  method: 'GET',
  // Browser negotiates HTTP/3 via Alt-Svc header
});

console.log(response.status); // Works transparently
```

### HTTP/3 Adoption Status (2024-2025)

**Browsers:**
- Chrome: Full support
- Firefox: Full support
- Safari: Full support
- Edge: Full support

**Servers:**
- Cloudflare: Full deployment
- Google: Full deployment
- Facebook: Full deployment
- AWS CloudFront: Support available
- nginx: Experimental support
- Node.js: Limited support

**Challenges:**
- Corporate firewalls often block UDP
- Some ISPs throttle UDP traffic
- Debugging tools less mature than for TCP

---

## Part VII: The Final Comparison

### When to Use What

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| **Transport** | HTTP/1.1 or HTTP/2 | HTTP (usually) | HTTP/2 required |
| **Payload Format** | JSON (usually) | JSON | Protocol Buffers (binary) |
| **Schema** | Optional (OpenAPI) | Required | Required (.proto) |
| **Client Coupling** | Low | Medium | High |
| **Performance** | Medium | Variable | High |
| **Streaming** | Hard (SSE) | Limited (subscriptions) | Native |
| **Browser Support** | Excellent | Excellent | Limited (needs grpc-web) |
| **Debugging** | Easy (curl, browser) | Medium (GraphiQL) | Harder (needs tools) |
| **Best For** | Public APIs, microservices | Flexible client apps | Internal service-to-service |

### REST: When to Choose It

**Use REST when:**
- Building public APIs (universal compatibility)
- Clients are diverse (web, mobile, IoT)
- Human readability matters
- Caching is critical (CDN-friendly)
- You want loose coupling

**Example:** Stripe API, GitHub API, Twitter API

### GraphQL: When to Choose It

**Use GraphQL when:**
- Frontend needs flexible data fetching
- You want to avoid over/under-fetching
- Mobile apps need to minimize requests
- You have complex, nested data relationships
- You can invest in proper batching/caching

**Example:** GitHub GraphQL API, Shopify API, Facebook

### gRPC: When to Choose It

**Use gRPC when:**
- Building internal service-to-service communication
- Performance is critical
- You need bidirectional streaming
- Type safety is important
- All services are under your control

**Example:** Google internal services, Netflix microservices

---

## Part VIII: Key Takeaways

### If You Remember Nothing Else

**REST is simple and universal.**
- Easy to debug, works everywhere
- But verbose and repetitive

**GraphQL is powerful but sharp.**
- Solves real problems with flexible queries
- But easy to get wrong (N+1, deep nesting)

**gRPC is fast but strict.**
- Excellent for service-to-service
- But requires HTTP/2 and tooling

**HTTP/3 is the future.**
- Solves TCP head-of-line blocking
- Faster connection establishment
- Connection migration support
- But adoption still growing

### The Real Understanding

If you understand:
- **CRLF** and why HTTP/1.1 parsing works the way it does
- **Headers** and why HTTP/2's compression matters
- **Batching** and why GraphQL can be slow without it
- **Stubs** and how gRPC hides networking complexity
- **Streams** and why they're natural in gRPC but awkward in REST
- **QUIC** and why UDP-based transport solves TCP's limitations
- **Protobuf encoding** and why binary is more efficient than text

Then you understand **real backend systems**.

### Production Checklist

**For REST:**
- ✅ Use HTTP/2 for better performance
- ✅ Implement rate limiting
- ✅ Add proper caching headers
- ✅ Version your API (v1, v2)
- ✅ Use pagination for large collections
- ✅ Implement CORS properly

**For GraphQL:**
- ✅ Implement DataLoader for N+1 prevention
- ✅ Add query depth limiting
- ✅ Implement query cost analysis
- ✅ Use persisted queries in production
- ✅ Add proper error handling
- ✅ Implement field-level authorization

**For gRPC:**
- ✅ Use client-side load balancing
- ✅ Implement health checks
- ✅ Add request deadlines
- ✅ Use interceptors for auth/logging
- ✅ Version your protobufs carefully
- ✅ Monitor stream health

---

## Part IX: Wireshark Packet Analysis

### Setting Up Wireshark for API Protocol Analysis

**Installation and Configuration:**

```bash
# Install Wireshark
# macOS
brew install wireshark

# Ubuntu/Debian
sudo apt-get install wireshark

# Enable capture for non-root users (Linux)
sudo usermod -a -G wireshark $USER
```

**Key Wireshark Filters:**

```
# HTTP/1.1 traffic
http

# HTTP/2 traffic
http2

# gRPC traffic (HTTP/2 with content-type)
http2 and (http2.header.content-type contains "application/grpc")

# QUIC/HTTP/3 traffic
quic

# Filter by host
http.host == "api.example.com"

# Filter by method
http.request.method == "POST"

# Filter by status code
http.response.code == 200
```

### Analyzing REST/HTTP/1.1 Packets

**Capture Filter:**
```
tcp port 80 or tcp port 443
```

**Sample REST Request Analysis:**

```
Frame 1: 180 bytes on wire
Ethernet II, Src: 00:1a:2b:3c:4d:5e, Dst: 00:5e:4d:3c:2b:1a
Internet Protocol Version 4, Src: 192.168.1.100, Dst: 93.184.216.34
Transmission Control Protocol, Src Port: 54321, Dst Port: 80
Hypertext Transfer Protocol
    POST /api/users HTTP/1.1\r\n
    Host: api.example.com\r\n
    User-Agent: curl/7.68.0\r\n
    Accept: */*\r\n
    Content-Type: application/json\r\n
    Content-Length: 27\r\n
    \r\n
    [Full request URI: http://api.example.com/api/users]
    [HTTP request 1/1]
    File Data: 27 bytes
    
Line-based text data: application/json (1 lines)
    {"name":"John","id":123}\n
```

**Key Observations:**
- Each header clearly visible in plaintext
- `\r\n` separators shown explicitly
- Total overhead: ~150 bytes of headers for 27 bytes of data
- Single TCP stream per request

**Response Analysis:**

```
Frame 2: 245 bytes on wire
Hypertext Transfer Protocol
    HTTP/1.1 201 Created\r\n
    Date: Sat, 27 Dec 2025 10:30:00 GMT\r\n
    Server: nginx/1.18.0\r\n
    Content-Type: application/json\r\n
    Content-Length: 87\r\n
    Connection: keep-alive\r\n
    X-Request-ID: abc123\r\n
    \r\n
    [HTTP response 1/1]
    
Line-based text data: application/json
    {"id":123,"name":"John","email":"john@example.com","created_at":"2025-12-27T10:30:00Z"}
```

### Analyzing HTTP/2 Packets

**Wireshark HTTP/2 Dissector Settings:**

```
Preferences → Protocols → HTTP2
☑ Reassemble HTTP2 headers spanning multiple frames
☑ Reassemble HTTP2 body spanning multiple frames
```

**Sample HTTP/2 Stream:**

```
Frame 1: TCP SYN (Connection establishment)
Frame 2: TCP SYN-ACK
Frame 3: TCP ACK
Frame 4: TLS Client Hello
Frame 5: TLS Server Hello
... TLS Handshake ...
Frame 10: HTTP/2 Connection Preface
    PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
    
Frame 11: HTTP/2 SETTINGS Frame
    Stream: 0 (connection-level)
    Length: 18
    Type: SETTINGS (4)
    Flags: 0x00
    Settings:
        SETTINGS_HEADER_TABLE_SIZE: 4096
        SETTINGS_ENABLE_PUSH: 1
        SETTINGS_MAX_CONCURRENT_STREAMS: 100

Frame 12: HTTP/2 HEADERS Frame (Stream 1)
    Length: 45
    Type: HEADERS (1)
    Flags: 0x05 (END_HEADERS, END_STREAM)
    Stream ID: 1
    Header Block Fragment:
        :method: GET
        :path: /api/users/123
        :scheme: https
        :authority: api.example.com
        user-agent: MyApp/1.0
        accept: application/json
    [Encoded using HPACK]
    [Actual bytes: 82 84 86 41 8a ... (binary)]

Frame 13: HTTP/2 HEADERS Frame (Stream 1 - Response)
    Length: 38
    Type: HEADERS (1)
    Flags: 0x04 (END_HEADERS)
    Stream ID: 1
    :status: 200
    content-type: application/json
    content-length: 87

Frame 14: HTTP/2 DATA Frame (Stream 1)
    Length: 87
    Type: DATA (0)
    Flags: 0x01 (END_STREAM)
    Stream ID: 1
    Data: {"id":123,"name":"John"...}
```

**Key Observations:**
- Multiple streams multiplexed on one connection
- Headers compressed using HPACK (binary format)
- Stream ID identifies which request/response
- Much less overhead than HTTP/1.1
- Can see concurrent streams (1, 3, 5, 7) interleaved

**HPACK Compression Visualization:**

```
First Request (Stream 1):
Frame bytes: 82 84 86 41 8A ...
Decoded:
    82 = Index 2 (:method: GET)
    84 = Index 4 (:path: /)
    86 = Index 6 (:scheme: https)
    41 8A ... = Literal header with indexing

Second Request (Stream 3):
Frame bytes: 82 84 BE ...  (Much shorter!)
Decoded:
    82 = Index 2 (:method: GET) 
    84 = Index 4 (:path: /)
    BE = Index 62 (cached authorization header)
    
Saved ~200 bytes!
```

### Analyzing gRPC Packets

**Wireshark gRPC Setup:**

```
Preferences → Protocols → ProtoBuf
☑ Load .proto files
☑ Dissect Protobuf fields as text

Add proto search path: /path/to/your/protos
```

**Sample gRPC Call:**

```
Frame 1: HTTP/2 HEADERS (Stream 1 - gRPC Request)
    :method: POST
    :scheme: https
    :path: /user.v1.UserService/GetUser
    :authority: api.example.com:50051
    content-type: application/grpc+proto
    te: trailers
    grpc-timeout: 5S
    grpc-encoding: gzip

Frame 2: HTTP/2 DATA (Stream 1 - gRPC Message)
    Length: 10
    gRPC Message:
        Compressed Flag: 0 (Not compressed)
        Message Length: 5 bytes
        Protobuf Message:
            Field 1 (id): varint = 123
            [Hex: 08 7B]
            
With .proto loaded, Wireshark shows:
    GetUserRequest {
        id: 123
    }

Frame 3: HTTP/2 HEADERS (Stream 1 - gRPC Response Headers)
    :status: 200
    content-type: application/grpc+proto
    grpc-encoding: identity

Frame 4: HTTP/2 DATA (Stream 1 - gRPC Response Message)
    Length: 45
    gRPC Message:
        Compressed Flag: 0
        Message Length: 40 bytes
        Protobuf Message:
            Field 1 (id): varint = 123
            Field 2 (name): length-delimited = "John"
            Field 3 (email): length-delimited = "john@example.com"
            
With .proto:
    GetUserResponse {
        user: {
            id: 123
            name: "John"
            email: "john@example.com"
        }
    }

Frame 5: HTTP/2 HEADERS (Stream 1 - gRPC Trailers)
    Flags: END_STREAM
    grpc-status: 0  (OK)
    grpc-message: ""
```

**Key Observations:**
- Uses HTTP/2 as transport
- Binary protobuf payload
- 5-byte gRPC frame wrapper
- Status sent in trailers (after data)
- Much more compact than JSON

**Comparing Sizes:**

```
REST/JSON:
Headers: ~250 bytes
Body: {"id":123,"name":"John","email":"john@example.com"} = 57 bytes
Total: ~307 bytes

gRPC/Protobuf:
Headers: ~150 bytes (compressed)
Body: 08 7B 12 04 4A 6F 68 6E 1A 11 6A 6F... = 40 bytes
Total: ~190 bytes

Savings: ~38%
```

### Analyzing HTTP/3/QUIC Packets

**Wireshark QUIC Configuration:**

```
Preferences → Protocols → QUIC
☑ Reassemble QUIC Stream
☑ Try to decrypt QUIC traffic
Add TLS key log file: /path/to/keys.log
```

**Sample HTTP/3 Exchange:**

```
Frame 1: QUIC Initial Packet
    QUIC Connection ID: abc123xyz789
    Packet Number: 0
    Version: 1 (0x00000001)
    Token Length: 0
    Length: 1200
    Payload:
        CRYPTO Frame (TLS ClientHello)
            TLS Version: 1.3
            Cipher Suites
            Extensions (ALPN: h3)

Frame 2: QUIC Handshake Packet
    Connection ID: abc123xyz789
    Packet Number: 1
    CRYPTO Frame (TLS ServerHello, Certificate, Finished)

Frame 3: QUIC 1-RTT Packet (Stream 0)
    Connection ID: abc123xyz789
    Packet Number: 2
    STREAM Frame:
        Stream ID: 0 (Control Stream)
        HTTP/3 SETTINGS Frame
        
Frame 4: QUIC 1-RTT Packet (Stream 4)
    Connection ID: abc123xyz789
    Packet Number: 3
    STREAM Frame:
        Stream ID: 4 (Request Stream)
        HTTP/3 HEADERS Frame:
            :method: GET
            :path: /api/users/123
            :scheme: https
            :authority: api.example.com

Frame 5: QUIC 1-RTT Packet (Stream 4 - Response)
    STREAM Frame:
        Stream ID: 4
        HTTP/3 HEADERS Frame:
            :status: 200
        HTTP/3 DATA Frame:
            Length: 87
            Data: {"id":123...}

Frame 6: QUIC 1-RTT Packet (ACK)
    ACK Frame:
        Largest Acknowledged: 5
        ACK Delay: 50 microseconds
```

**Key Observations:**
- All encrypted (needs key log for decryption)
- UDP datagrams instead of TCP segments
- Independent stream recovery (no head-of-line blocking)
- Connection ID survives IP migration
- Faster handshake (1-RTT or 0-RTT)

**Packet Loss Demonstration:**

```
HTTP/2 over TCP:
Packet 100: Stream 1 data
Packet 101: Stream 3 data (LOST!)
Packet 102: Stream 5 data
Result: All streams wait for retransmit of packet 101

HTTP/3 over QUIC:
Packet 100: Stream 1 data
Packet 101: Stream 3 data (LOST!)
Packet 102: Stream 5 data (delivered!)
Result: Only Stream 3 waits for retransmit
```

### Practical Wireshark Commands

**Export HTTP Objects:**
```
File → Export Objects → HTTP
# Extract all JSON payloads, images, etc.
```

**Follow TCP/HTTP2 Stream:**
```
Right-click packet → Follow → TCP Stream
Right-click packet → Follow → HTTP/2 Stream
```

**Statistics:**
```
Statistics → HTTP → Requests
Statistics → HTTP2 → Stream Analysis
```

**Time Analysis:**
```
Statistics → Flow Graph
# Visual timeline of request/response
```

---

## Part X: HTTP/3 Deployment Strategies

### Strategy 1: Alt-Svc Header (Gradual Migration)

**How It Works:**

```
Client                          Server
  |                               |
  |-- HTTP/2 Request ------------>|
  |                               |
  |<-- HTTP/2 Response -----------|
  |    Alt-Svc: h3=":443"        |
  |                               |
  |-- HTTP/3 Request (UDP:443)-->|
  |                               |
  |<-- HTTP/3 Response -----------|
```

**Server Configuration (nginx):**

```nginx
server {
    listen 443 ssl http2;
    listen 443 quic reuseport;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # Enable HTTP/3
    http3 on;
    
    # Advertise HTTP/3 support
    add_header Alt-Svc 'h3=":443"; ma=86400';
    
    location / {
        proxy_pass http://backend;
    }
}
```

**Cloudflare Configuration:**

```
# Cloudflare dashboard
Speed → Optimization → HTTP/3 (with QUIC) → Enable

# Or via API
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/{zone_id}/settings/http3" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"value":"on"}'
```

### Strategy 2: DNS SRV Records

**DNS Configuration:**

```dns
; Advertise HTTP/3 support via DNS
_https._udp.example.com. 3600 IN SRV 0 1 443 server.example.com.

; Traditional A record
example.com. 3600 IN A 93.184.216.34
```

### Strategy 3: Dual-Stack Deployment

**Load Balancer Configuration (HAProxy):**

```haproxy
# HTTP/1.1 and HTTP/2
frontend http_frontend
    bind *:443 ssl crt /path/to/cert.pem alpn h2,http/1.1
    default_backend http_backend

# HTTP/3 (separate listener)
frontend http3_frontend
    bind *:443 quic ssl crt /path/to/cert.pem alpn h3
    default_backend http_backend

backend http_backend
    balance roundrobin
    server app1 10.0.1.10:8080 check
    server app2 10.0.1.11:8080 check
```

### Strategy 4: Progressive Enhancement

**Client Implementation:**

```javascript
class AdaptiveHttpClient {
  constructor(baseURL) {
    this.baseURL = baseURL;
    this.http3Supported = null;
  }
  
  async fetch(path, options) {
    // Try HTTP/3 if previously successful
    if (this.http3Supported === true) {
      try {
        return await this.fetchHTTP3(path, options);
      } catch (error) {
        console.warn('HTTP/3 failed, falling back to HTTP/2');
        this.http3Supported = false;
      }
    }
    
    // Try HTTP/2 with Alt-Svc detection
    const response = await fetch(`${this.baseURL}${path}`, options);
    
    // Check for HTTP/3 advertisement
    const altSvc = response.headers.get('alt-svc');
    if (altSvc && altSvc.includes('h3=')) {
      this.http3Supported = true;
    }
    
    return response;
  }
  
  async fetchHTTP3(path, options) {
    // Use HTTP/3 endpoint
    return fetch(`${this.baseURL}${path}`, {
      ...options,
      // Browser handles HTTP/3 automatically
    });
  }
}
```

### Monitoring HTTP/3 Adoption

**Server-Side Metrics:**

```javascript
import prometheus from 'prom-client';

const http3RequestsCounter = new prometheus.Counter({
  name: 'http3_requests_total',
  help: 'Total HTTP/3 requests',
  labelNames: ['status', 'method']
});

const protocolGauge = new prometheus.Gauge({
  name: 'active_connections_by_protocol',
  help: 'Active connections by protocol',
  labelNames: ['protocol']
});

app.use((req, res, next) => {
  const protocol = req.httpVersion === '3.0' ? 'http3' :
                   req.httpVersion === '2.0' ? 'http2' : 'http1.1';
  
  protocolGauge.inc({ protocol });
  
  res.on('finish', () => {
    if (protocol === 'http3') {
      http3RequestsCounter.inc({
        status: res.statusCode,
        method: req.method
      });
    }
    protocolGauge.dec({ protocol });
  });
  
  next();
});
```

### Common HTTP/3 Deployment Issues

**Issue 1: UDP Blocked by Firewall**

```bash
# Test UDP connectivity
nc -u -v api.example.com 443

# Check if UDP 443 is open
nmap -sU -p 443 api.example.com
```

**Solution:**
```nginx
# Keep HTTP/2 as fallback
listen 443 ssl http2;  # TCP fallback
listen 443 quic;       # HTTP/3
```

**Issue 2: Load Balancer Doesn't Support QUIC**

**Solution:** Use Layer 7 LB or upgrade:

```
Client → CDN (HTTP/3) → Origin (HTTP/2)
         Cloudflare          Your servers
```

**Issue 3: Connection Migration Not Working**

**Debug:**
```javascript
// Log connection ID changes
connection.on('migration', (newAddr, oldAddr) => {
  console.log(`Connection migrated from ${oldAddr} to ${newAddr}`);
});
```

---

## Part XI: Advanced GraphQL Federation

### What is GraphQL Federation?

Federation allows you to split a monolithic GraphQL schema across multiple services, each owning its own data.

**Monolithic GraphQL:**
```
Client → Single GraphQL Server → Multiple databases
```

**Federated GraphQL:**
```
Client → Gateway → User Service (users data)
                 → Order Service (orders data)
                 → Product Service (products data)
```

### Federation Architecture

```
                     Apollo Gateway
                           |
        +------------------+------------------+
        |                  |                  |
   User Service      Order Service     Product Service
   (Port 4001)       (Port 4002)       (Port 4003)
        |                  |                  |
    Users DB          Orders DB         Products DB
```

### Implementing Federation: User Service

**Schema (users-service.graphql):**

```graphql
extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.3")

type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
  # This service owns these fields
}

type Query {
  user(id: ID!): User
  users: [User!]!
}
```

**Resolver:**

```javascript
import { buildSubgraphSchema } from '@apollo/subgraph';
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';
import gql from 'graphql-tag';

const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.3")

  type User @key(fields: "id") {
    id: ID!
    name: String!
    email: String!
  }

  type Query {
    user(id: ID!): User
    users: [User!]!
  }
`;

const resolvers = {
  Query: {
    user: (_, { id }) => {
      return db.users.findByPk(id);
    },
    users: () => {
      return db.users.findAll();
    }
  },
  User: {
    // Entity resolver for federation
    __resolveReference: (reference) => {
      return db.users.findByPk(reference.id);
    }
  }
};

const server = new ApolloServer({
  schema: buildSubgraphSchema({ typeDefs, resolvers })
});

const { url } = await startStandaloneServer(server, {
  listen: { port: 4001 }
});
```

### Implementing Federation: Order Service

**Schema (orders-service.graphql):**

```graphql
extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.3")

type Order @key(fields: "id") {
  id: ID!
  total: Float!
  status: OrderStatus!
  # Reference to User service
  user: User!
}

# Extend User type from another service
extend type User @key(fields: "id") {
  id: ID! @external
  # Add orders field to User
  orders: [Order!]!
}

enum OrderStatus {
  PENDING
  COMPLETED
  CANCELLED
}

type Query {
  order(id: ID!): Order
}
```

**Resolver:**

```javascript
const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.3")

  type Order @key(fields: "id") {
    id: ID!
    total: Float!
    status: OrderStatus!
    user: User!
  }

  extend type User @key(fields: "id") {
    id: ID! @external
    orders: [Order!]!
  }

  enum OrderStatus {
    PENDING
    COMPLETED
    CANCELLED
  }

  type Query {
    order(id: ID!): Order
  }
`;

const resolvers = {
  Query: {
    order: (_, { id }) => {
      return db.orders.findByPk(id);
    }
  },
  Order: {
    __resolveReference: (reference) => {
      return db.orders.findByPk(reference.id);
    },
    user: (order) => {
      // Return reference to User
      return { __typename: 'User', id: order.userId };
    }
  },
  User: {
    // Extend User with orders field
    orders: (user) => {
      return db.orders.findAll({
        where: { userId: user.id }
      });
    }
  }
};

const server = new ApolloServer({
  schema: buildSubgraphSchema({ typeDefs, resolvers })
});

await startStandaloneServer(server, { listen: { port: 4002 } });
```

### Federation Gateway

**Gateway Configuration:**

```javascript
import { ApolloGateway, IntrospectAndCompose } from '@apollo/gateway';
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: 'users', url: 'http://localhost:4001/graphql' },
      { name: 'orders', url: 'http://localhost:4002/graphql' },
      { name: 'products', url: 'http://localhost:4003/graphql' }
    ],
    // Polling interval for schema updates
    pollIntervalInMs: 10000
  })
});

const server = new ApolloServer({
  gateway,
  // Disable subscriptions (not yet supported in federation)
  subscriptions: false
});

const { url } = await startStandaloneServer(server, {
  listen: { port: 4000 }
});

console.log(`🚀 Gateway ready at ${url}`);
```

### Federated Query Execution

**Client Query:**

```graphql
query GetUserWithOrders {
  user(id: "123") {
    id
    name        # From users service
    email       # From users service
    orders {    # From orders service
      id
      total
      status
    }
  }
}
```

**Execution Plan:**

```
1. Gateway receives query
2. Parse and plan:
   - user(id: "123") { name, email } → users service
   - User.orders → orders service
   
3. Execute:
   Step 1: Query users service
   POST http://localhost:4001/graphql
   {
     query {
       _entities(representations: [{__typename: "User", id: "123"}]) {
         ... on User {
           id
           name
           email
         }
       }
     }
   }
   
   Step 2: Query orders service (with user reference)
   POST http://localhost:4002/graphql
   {
     query {
       _entities(representations: [{__typename: "User", id: "123"}]) {
         ... on User {
           orders {
             id
             total
             status
           }
         }
       }
     }
   }
   
4. Merge results:
   {
     "data": {
       "user": {
         "id": "123",
         "name": "John",      // From users service
         "email": "...",       // From users service
         "orders": [...]       // From orders service
       }
     }
   }
```

### Advanced Federation Patterns

**Pattern 1: Value Types (Shared Types)**

```graphql
# Common types shared across services
scalar DateTime

type Address {
  street: String!
  city: String!
  country: String!
}

# Each service can use Address without @key
type User {
  id: ID!
  address: Address
}

type Warehouse {
  id: ID!
  address: Address
}
```

**Pattern 2: Interfaces in Federation**

```graphql
# products-service
interface Node {
  id: ID!
}

type Product implements Node @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
}

# orders-service extends the interface
extend interface Node {
  id: ID!
}

type Order implements Node @key(fields: "id") {
  id: ID!
  total: Float!
}
```

**Pattern 3: Computed Fields Across Services**

```graphql
# orders-service
extend type User @key(fields: "id") {
  id: ID! @external
  # Computed field using data from multiple services
  totalSpent: Float! @requires(fields: "orders { total }")
}
```

**Resolver:**

```javascript
User: {
  totalSpent: (user) => {
    return user.orders.reduce((sum, order) => sum + order.total, 0);
  }
}
```

### Federation Error Handling

```javascript
import { ApolloGateway } from '@apollo/gateway';

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [...]
  }),
  
  // Custom error formatting
  formatError: (error) => {
    console.error('Gateway error:', error);
    
    // Hide internal errors from clients
    if (error.extensions?.code === 'INTERNAL_SERVER_ERROR') {
      return new GraphQLError('An error occurred', {
        extensions: {
          code: 'INTERNAL_SERVER_ERROR'
        }
      });
    }
    
    return error;
  },
  
  // Service health checks
  serviceHealthCheck: true,
  
  // Fallback behavior
  experimental_didResolveSubgraphResponse: async (options) => {
    if (options.response.errors) {
      console.error(`Errors from ${options.subgraphName}:`, 
                    options.response.errors);
    }
  }
});
```

### Federation Performance Optimization

**Batching Entity Resolves:**

```javascript
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (references) => {
  const ids = references.map(ref => ref.id);
  const users = await db.users.findAll({ where: { id: ids } });
  
  const userMap = new Map(users.map(u => [u.id, u]));
  return references.map(ref => userMap.get(ref.id));
});

const resolvers = {
  User: {
    __resolveReference: (reference, context) => {
      return context.loaders.userLoader.load(reference);
    }
  }
};
```

---

## Part XII: gRPC-Web for Browser Clients

### Why gRPC-Web Exists

Browsers can't make native gRPC calls because:
- No direct HTTP/2 control
- Can't set required HTTP/2 headers
- No access to trailers before body

**Solution:** gRPC-Web = gRPC adapted for browsers

### gRPC-Web Architecture

```
Browser Client
      ↓
  gRPC-Web (HTTP/1.1 or HTTP/2)
      ↓
  Envoy Proxy (translation layer)
      ↓
  gRPC Server (HTTP/2)
```

### Setting Up Envoy Proxy

**envoy.yaml:**

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 8080
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: grpc_web
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: grpc_backend
                  timeout: 60s
                  max_stream_duration:
                    grpc_timeout_header_max: 60s
              cors:
                allow_origin_string_match:
                - prefix: "*"
                allow_methods: GET, PUT, DELETE, POST, OPTIONS
                allow_headers: keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout
                max_age: "1728000"
                expose_headers: grpc-status,grpc-message
          http_filters:
          - name: envoy.filters.http.grpc_web
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
          - name: envoy.filters.http.cors
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: grpc_backend
    type: LOGICAL_DNS
    lb_policy: ROUND_ROBIN
    dns_lookup_family: V4_ONLY
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: grpc_backend
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: localhost
                port_value: 50051
```

**Start Envoy:**

```bash
docker run -d -p 8080:8080 \
  -v $(pwd)/envoy.yaml:/etc/envoy/envoy.yaml \
  envoyproxy/envoy:v1.28-latest
```

### Client Implementation (JavaScript)

**Install Dependencies:**

```bash
npm install grpc-web
npm install google-protobuf
npm install --save-dev grpc-tools
```

**Generate Web Client Code:**

```bash
# Generate JS code from proto
protoc -I=. user.proto \
  --js_out=import_style=commonjs:./src/generated \
  --grpc-web_out=import_style=commonjs,mode=grpcwebtext:./src/generated
```

**Client Code:**

```javascript
import { UserServiceClient } from './generated/user_grpc_web_pb';
import { GetUserRequest } from './generated/user_pb';

// Create client pointing to Envoy proxy
const client = new UserServiceClient('http://localhost:8080', null, null);

// Unary call
function getUser(userId) {
  const request = new GetUserRequest();
  request.setId(userId);
  
  return new Promise((resolve, reject) => {
    client.getUser(request, {}, (err, response) => {
      if (err) {
        console.error('Error:', err.code, err.message);
        reject(err);
      } else {
        const user = {
          id: response.getUser().getId(),
          name: response.getUser().getName(),
          email: response.getUser().getEmail()
        };
        resolve(user);
      }
    });
  });
}

// Server streaming
function listOrders(userId) {
  const request = new ListOrdersRequest();
  request.setUserId(userId);
  
  const stream = client.listUserOrders(request, {});
  
  stream.on('data', (response) => {
    console.log('Received order:', {
      id: response.getId(),
      total: response.getTotal(),
      status: response.getStatus()
    });
  });
  
  stream.on('status', (status) => {
    console.log('Stream status:', status.code, status.details);
  });
  
  stream.on('end', () => {
    console.log('Stream ended');
  });
}

// Usage in React component
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [orders, setOrders] = useState([]);
  
  useEffect(() => {
    // Fetch user
    getUser(userId).then(setUser);
    
    // Stream orders
    const request = new ListOrdersRequest();
    request.setUserId(userId);
    const stream = client.listUserOrders(request, {});
    
    stream.on('data', (order) => {
      setOrders(prev => [...prev, {
        id: order.getId(),
        total: order.getTotal()
      }]);
    });
    
    return () => stream.cancel();
  }, [userId]);
  
  return (
    <div>
      <h1>{user?.name}</h1>
      <ul>
        {orders.map(order => (
          <li key={order.id}>Order ${order.total}</li>
        ))}
      </ul>
    </div>
  );
}
```

### gRPC-Web vs gRPC Comparison

| Feature | gRPC (native) | gRPC-Web |
|---------|---------------|----------|
| Transport | HTTP/2 only | HTTP/1.1 or HTTP/2 |
| Trailers | Full support | Limited (text mode) |
| Client streaming | Yes | No |
| Bidirectional streaming | Yes | No |
| Server streaming | Yes | Yes |
| Compression | gzip, deflate | gzip only |
| Browser support | No | Yes |

### Advanced: TypeScript with gRPC-Web

**Generate TypeScript definitions:**

```bash
protoc -I=. user.proto \
  --js_out=import_style=commonjs:./src/generated \
  --grpc-web_out=import_style=typescript,mode=grpcwebtext:./src/generated
```

**Type-safe client:**

```typescript
import { UserServiceClient } from './generated/UserServiceClientPb';
import { GetUserRequest, User } from './generated/user_pb';
import { Error as GrpcError } from 'grpc-web';

class UserClient {
  private client: UserServiceClient;
  
  constructor(baseUrl: string) {
    this.client = new UserServiceClient(baseUrl, null, null);
  }
  
  async getUser(id: string): Promise<User> {
    const request = new GetUserRequest();
    request.setId(id);
    
    return new Promise((resolve, reject) => {
      this.client.getUser(request, {}, (err: GrpcError, response) => {
        if (err) {
          reject(this.handleError(err));
        } else {
          resolve(response.getUser()!);
        }
      });
    });
  }
  
  private handleError(err: GrpcError): Error {
    switch (err.code) {
      case 5: // NOT_FOUND
        return new Error('User not found');
      case 16: // UNAUTHENTICATED
        return new Error('Authentication required');
      default:
        return new Error(err.message);
    }
  }
}
```

---

## Part XIII: Protocol Buffers Schema Evolution

### Schema Evolution Basics

Protocol Buffers allows you to evolve your schema over time without breaking existing clients.

**Rules for Safe Evolution:**

1. **Never change field numbers**
2. **Never change field types** (with exceptions)
3. **Can add new fields**
4. **Can remove optional fields**
5. **Use reserved for deleted fields**

### Version 1: Initial Schema

```protobuf
syntax = "proto3";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

### Version 2: Adding Fields (Safe)

```protobuf
syntax = "proto3";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  
  // New fields - old clients ignore them
  string phone = 4;
  google.protobuf.Timestamp created_at = 5;
  UserRole role = 6;
}

enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_ADMIN = 1;
  USER_ROLE_USER = 2;
}
```

**Backward compatibility:**
- Old clients can read new messages (ignore unknown fields)
- New clients can read old messages (missing fields use defaults)

### Version 3: Removing Fields (Safe with Reservation)

```protobuf
syntax = "proto3";

message User {
  int64 id = 1;
  string name = 2;
  // email removed - field number reserved
  reserved 3;
  reserved "email";
  
  string phone = 4;
  google.protobuf.Timestamp created_at = 5;
  UserRole role = 6;
  
  // New field reusing higher numbers
  string username = 7;
}
```

**Why reserve?**
- Prevents accidental reuse of field numbers
- Prevents accidental reuse of field names

### Version 4: Nested Messages

```protobuf
syntax = "proto3";

message User {
  int64 id = 1;
  string name = 2;
  reserved 3;
  string phone = 4;
  google.protobuf.Timestamp created_at = 5;
  UserRole role = 6;
  string username = 7;
  
  // New nested message
  Address address = 8;
  repeated SocialLink social_links = 9;
}

message Address {
  string street = 1;
  string city = 2;
  string country = 3;
  string postal_code = 4;
}

message SocialLink {
  SocialPlatform platform = 1;
  string url = 2;
}

enum SocialPlatform {
  SOCIAL_PLATFORM_UNSPECIFIED = 0;
  SOCIAL_PLATFORM_TWITTER = 1;
  SOCIAL_PLATFORM_LINKEDIN = 2;
  SOCIAL_PLATFORM_GITHUB = 3;
}
```

### Safe Type Changes

**These changes are safe:**

```protobuf
// Can change between these groups:
int32, uint32, int64, uint64, bool
sint32, sint64
string, bytes

// Example: Extending integer type
message User {
  // v1: int32 id = 1;
  int64 id = 1;  // Safe: int32 → int64
}
```

**These changes are UNSAFE:**

```protobuf
// ❌ Unsafe changes:
int64 → string     // Different wire types
string → int64
int32 → float
repeated → single value
message → scalar
```

### Handling Breaking Changes

**Strategy 1: New Field Number**

```protobuf
message User {
  reserved 2;
  reserved "name";
  
  // Old field
  // string name = 2;
  
  // New field with different type
  FullName full_name = 7;
}

message FullName {
  string first_name = 1;
  string last_name = 2;
}
```

**Strategy 2: Wrapper Types**

```protobuf
import "google/protobuf/wrappers.proto";

message User {
  int64 id = 1;
  
  // Allows distinguishing between unset and zero
  google.protobuf.Int32Value age = 2;
  google.protobuf.StringValue nickname = 3;
}
```

**Strategy 3: Oneof for Type Changes**

```protobuf
message User {
  int64 id = 1;
  
  oneof identifier {
    string email = 2;        // Old way
    string username = 3;     // New way
    int64 user_number = 4;   // Alternative
  }
}
```

### Versioning Strategies

**Strategy 1: Package Versioning**

```protobuf
// v1/user.proto
syntax = "proto3";
package user.v1;

message User {
  int64 id = 1;
  string name = 2;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

```protobuf
// v2/user.proto
syntax = "proto3";
package user.v2;

message User {
  int64 id = 1;
  FullName name = 2;  // Breaking change
  string email = 3;
}

message FullName {
  string first_name = 1;
  string last_name = 2;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

**Server supports both:**

```javascript
// Register both versions
server.addService(v1.UserService.service, v1Handlers);
server.addService(v2.UserService.service, v2Handlers);

// Clients specify version
const clientV1 = new v1.UserServiceClient(...);
const clientV2 = new v2.UserServiceClient(...);
```

**Strategy 2: Feature Flags**

```protobuf
message User {
  int64 id = 1;
  string name = 2;
  
  // New field behind feature flag
  string new_field = 3;
}
```

```javascript
const resolvers = {
  User: (user, context) => {
    const response = {
      id: user.id,
      name: user.name
    };
    
    // Only include new field for opted-in clients
    if (context.features.includes('NEW_FIELD')) {
      response.new_field = user.newField;
    }
    
    return response;
  }
};
```

### Testing Schema Compatibility

```javascript
import { testCompatibility } from './proto-tester';

describe('Proto compatibility', () => {
  it('v2 can read v1 messages', () => {
    const v1Message = new v1.User();
    v1Message.setId(123);
    v1Message.setName('John');
    
    const bytes = v1Message.serializeBinary();
    
    // v2 client reads v1 message
    const v2Message = v2.User.deserializeBinary(bytes);
    expect(v2Message.getId()).toBe(123);
    expect(v2Message.getName().getFirstName()).toBe('John');
  });
  
  it('v1 can read v2 messages', () => {
    const v2Message = new v2.User();
    v2Message.setId(123);
    const name = new v2.FullName();
    name.setFirstName('John');
    name.setLastName('Doe');
    v2Message.setName(name);
    
    const bytes = v2Message.serializeBinary();
    
    // v1 client reads v2 message (ignores unknown fields)
    const v1Message = v1.User.deserializeBinary(bytes);
    expect(v1Message.getId()).toBe(123);
  });
});
```

---

## Part XIV: Load Testing Methodologies

### REST API Load Testing

**Tool: Apache Bench (ab)**

```bash
# Simple test: 1000 requests, 10 concurrent
ab -n 1000 -c 10 http://localhost:3000/api/users

# With POST data
ab -n 1000 -c 10 -p payload.json -T application/json \
  http://localhost:3000/api/users

# With auth header
ab -n 1000 -c 10 -H "Authorization: Bearer token123" \
  http://localhost:3000/api/users
```

**Tool: wrk (Advanced)**

```bash
# Basic load test
wrk -t12 -c400 -d30s http://localhost:3000/api/users

# With Lua script for complex scenarios
wrk -t12 -c400 -d30s -s script.lua http://localhost:3000
```

**script.lua:**

```lua
-- Complex request pattern
counter = 0

request = function()
  counter = counter + 1
  path = "/api/users/" .. counter
  
  wrk.headers["Authorization"] = "Bearer token123"
  
  if counter % 3 == 0 then
    -- Every 3rd request is a POST
    wrk.method = "POST"
    wrk.body = '{"name":"User' .. counter .. '"}'
    wrk.headers["Content-Type"] = "application/json"
    return wrk.format(nil, "/api/users")
  else
    -- Other requests are GETs
    wrk.method = "GET"
    return wrk.format(nil, path)
  end
end

response = function(status, headers, body)
  if status ~= 200 then
    print("Error: " .. status)
  end
end
```

**Tool: k6 (Modern, Scriptable)**

```javascript
// k6-script.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp up to 200 users
    { duration: '5m', target: 200 },  // Stay at 200 users
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms
    http_req_failed: ['rate<0.01'],    // Error rate under 1%
  },
};

export default function () {
  // Test GET endpoint
  const getRes = http.get('http://localhost:3000/api/users/123', {
    headers: { 'Authorization': 'Bearer token123' },
  });
  
  check(getRes, {
    'status is 200': (r) => r.status === 200,
    'response time OK': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
  
  // Test POST endpoint
  const payload = JSON.stringify({
    name: 'Test User',
    email: 'test@example.com',
  });
  
  const postRes = http.post('http://localhost:3000/api/users', payload, {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer token123',
    },
  });
  
  check(postRes, {
    'status is 201': (r) => r.status === 201,
  });
  
  sleep(1);
}
```

**Run k6:**

```bash
k6 run k6-script.js

# With custom settings
k6 run --vus 100 --duration 10m k6-script.js

# Output results to InfluxDB
k6 run --out influxdb=http://localhost:8086/k6 k6-script.js
```

### GraphQL Load Testing

**Tool: artillery**

```yaml
# artillery-graphql.yml
config:
  target: "http://localhost:4000"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 300
      arrivalRate: 50
      name: "Sustained load"
    - duration: 120
      arrivalRate: 100
      name: "Peak load"
  processor: "./graphql-functions.js"

scenarios:
  - name: "Get user with orders"
    weight: 70
    flow:
      - post:
          url: "/graphql"
          json:
            query: |
              query GetUser($id: ID!) {
                user(id: $id) {
                  id
                  name
                  orders {
                    id
                    total
                  }
                }
              }
            variables:
              id: "{{ $randomNumber(1, 1000) }}"
          capture:
            - json: "$.data.user.id"
              as: "userId"
          expect:
            - statusCode: 200
            - contentType: json
            - hasProperty: data.user

  - name: "Create user mutation"
    weight: 30
    flow:
      - post:
          url: "/graphql"
          json:
            query: |
              mutation CreateUser($name: String!, $email: String!) {
                createUser(name: $name, email: $email) {
                  id
                  name
                }
              }
            variables:
              name: "User {{ $randomString() }}"
              email: "{{ $randomString() }}@example.com"
```

**Run artillery:**

```bash
artillery run artillery-graphql.yml

# Generate HTML report
artillery run --output report.json artillery-graphql.yml
artillery report report.json
```

**Testing GraphQL N+1 Issues:**

```javascript
// graphql-n1-test.js
import { ApolloClient, InMemoryCache, gql } from '@apollo/client';

const client = new ApolloClient({
  uri: 'http://localhost:4000/graphql',
  cache: new InMemoryCache()
});

// Test query that triggers N+1
const TRIGGER_N1_QUERY = gql`
  query {
    users {
      id
      name
      orders {        # This might cause N+1!
        id
        total
        items {       # Nested N+1!
          id
          product {
            id
            name
          }
        }
      }
    }
  }
`;

async function testN1Performance() {
  console.time('N+1 Query');
  
  const result = await client.query({
    query: TRIGGER_N1_QUERY
  });
  
  console.timeEnd('N+1 Query');
  console.log('Users fetched:', result.data.users.length);
}

testN1Performance();
```

### gRPC Load Testing

**Tool: ghz**

```bash
# Install ghz
go install github.com/bojand/ghz/cmd/ghz@latest

# Simple load test
ghz --insecure \
  --proto ./user.proto \
  --call user.v1.UserService/GetUser \
  -d '{"id": "123"}' \
  -c 100 \
  -n 10000 \
  localhost:50051

# With custom metadata (auth)
ghz --insecure \
  --proto ./user.proto \
  --call user.v1.UserService/GetUser \
  -d '{"id": "123"}' \
  -m '{"authorization": "Bearer token123"}' \
  -c 100 \
  -n 10000 \
  localhost:50051

# Test streaming RPC
ghz --insecure \
  --proto ./user.proto \
  --call user.v1.UserService/ListUserOrders \
  -d '{"user_id": "123", "limit": 50}' \
  -c 50 \
  -n 1000 \
  localhost:50051

# Output JSON report
ghz --insecure \
  --proto ./user.proto \
  --call user.v1.UserService/GetUser \
  -d '{"id": "123"}' \
  -c 100 \
  -n 10000 \
  -O json \
  -o results.json \
  localhost:50051
```

**Advanced: Custom ghz Script**

```javascript
// ghz-config.json
{
  "proto": "./user.proto",
  "call": "user.v1.UserService/GetUser",
  "insecure": true,
  "total": 10000,
  "concurrency": 100,
  "connections": 10,
  "duration": "60s",
  "timeout": "5s",
  "data": {
    "id": "{{.RequestNumber}}"
  },
  "metadata": {
    "authorization": "Bearer token123"
  },
  "host": "localhost:50051"
}
```

```bash
ghz --config ghz-config.json
```

### Comparative Load Test

**Test Setup:**

```bash
# Same endpoint, different protocols
# REST: http://localhost:3000/api/users/123
# GraphQL: http://localhost:4000/graphql
# gRPC: localhost:50051
```

**REST:**

```bash
wrk -t12 -c400 -d30s http://localhost:3000/api/users/123
```

**GraphQL:**

```bash
k6 run graphql-load-test.js
```

**gRPC:**

```bash
ghz --insecure \
  --proto ./user.proto \
  --call user.v1.UserService/GetUser \
  -d '{"id": "123"}' \
  -c 400 \
  -d 30s \
  localhost:50051
```

**Expected Results (Typical Microservice):**

| Protocol | Req/sec | P95 Latency | P99 Latency | Throughput |
|----------|---------|-------------|-------------|------------|
| REST | 5,000 | 80ms | 150ms | 2.5 MB/s |
| GraphQL | 4,500 | 95ms | 180ms | 2.8 MB/s |
| gRPC | 15,000 | 30ms | 60ms | 3.5 MB/s |

**Why gRPC is faster:**
- Binary encoding (smaller payload)
- HTTP/2 multiplexing
- Connection reuse
- Less parsing overhead

### Monitoring During Load Tests

**Prometheus Metrics:**

```javascript
import promClient from 'prom-client';

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latencies',
  labelNames: ['method', 'route', 'status_code']
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .observe(duration);
      
    httpRequestTotal
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .inc();
  });
  
  next();
});
```

**Grafana Dashboard Query:**

```promql
# Request rate
rate(http_requests_total[5m])

# P95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Error rate
rate(http_requests_total{status_code=~"5.."}[5m]) / 
rate(http_requests_total[5m])
```

---

## Part XV: Final Synthesis

### Decision Matrix

**Choose REST when:**
- ✅ Building public APIs
- ✅ Need maximum compatibility
- ✅ HTTP caching is critical
- ✅ Simple CRUD operations
- ✅ Documentation is priority (OpenAPI/Swagger)

**Choose GraphQL when:**
- ✅ Frontend needs flexible queries
- ✅ Multiple client types (web, mobile, desktop)
- ✅ Reducing over-fetching is important
- ✅ You have resources for proper implementation
- ✅ Complex, nested data relationships

**Choose gRPC when:**
- ✅ Internal service-to-service communication
- ✅ Performance is critical
- ✅ Type safety matters
- ✅ Streaming is required
- ✅ You control both client and server

**Choose HTTP/3 when:**
- ✅ Building for mobile clients (connection migration)
- ✅ High packet loss networks
- ✅ Need faster connection establishment
- ✅ Already using HTTP/2 (easy upgrade)

### Production Checklist

**REST:**
- ✅ Use HTTP/2
- ✅ Implement rate limiting
- ✅ Add caching headers
- ✅ Version your API
- ✅ Use pagination
- ✅ Implement CORS
- ✅ Add request tracing
- ✅ Monitor P95/P99 latencies

**GraphQL:**
- ✅ Implement DataLoader
- ✅ Add query depth limiting
- ✅ Implement cost analysis
- ✅ Use persisted queries
- ✅ Add error handling
- ✅ Field-level authorization
- ✅ Monitor resolver performance
- ✅ Set up APM tracing

**gRPC:**
- ✅ Client-side load balancing
- ✅ Health checks
- ✅ Request deadlines
- ✅ Interceptors for auth/logging
- ✅ Version protobufs carefully
- ✅ Monitor stream health
- ✅ Set up Envoy for gRPC-Web
- ✅ Test schema evolution

### Performance Optimization Summary

| Technique | REST | GraphQL | gRPC |
|-----------|------|---------|------|
| HTTP/2 | ✅ Major improvement | ✅ Standard | ✅ Required |
| Compression | gzip/brotli | gzip/brotli | Built-in |
| Batching | Manual | DataLoader | Built-in |
| Caching | HTTP cache | Complex | Custom |
| Streaming | SSE | Subscriptions | Native |

---

## Downloadable Resources

### Complete Code Repository Structure

```
api-protocols-guide/
├── rest/
│   ├── server-modern.js
│   ├── server-legacy.php
│   ├── client-modern.js
│   └── client-legacy.js
├── graphql/
│   ├── schema.graphql
│   ├── server-modern.js
│   ├── server-legacy.js
│   ├── resolvers.js
│   ├── dataloader.js
│   └── federation/
│       ├── gateway.js
│       ├── users-service.js
│       └── orders-service.js
├── grpc/
│   ├── proto/
│   │   └── user.proto
│   ├── server.js
│   ├── client.js
│   ├── load-balancer/
│   │   └── envoy.yaml
│   └── grpc-web/
│       ├── envoy.yaml
│       └── client.js
├── http3/
│   ├── server.js
│   └── config.yaml
├── load-testing/
│   ├── rest-test.js
│   ├── graphql-test.yml
│   ├── grpc-test.json
│   └── k6-script.js
├── wireshark/
│   └── filters.txt
└── monitoring/
    ├── prometheus.yml
    └── grafana-dashboard.json
```

### Key Takeaways

**If you remember nothing else:**

1. **CRLF** is fundamental to HTTP/1.1
2. **HTTP/2** uses binary frames and HPACK compression
3. **GraphQL** requires DataLoader to avoid N+1
4. **gRPC** uses Protobuf for compact binary encoding
5. **HTTP/3** solves TCP head-of-line blocking with QUIC
6. **Load balancing** differs significantly between protocols
7. **Schema evolution** requires careful planning
8. **Load testing** reveals performance characteristics

**The Real Understanding:**

Protocols are just **bytes on a wire**. The differences come from:
- How those bytes are formatted
- How connections are managed
- How errors are handled
- How data is compressed

Master these fundamentals and you'll understand not just today's protocols, but tomorrow's as well.

---

*This comprehensive guide covers everything from byte-level packet analysis to production deployment strategies. Use it as a reference when building and scaling modern API systems.*

---

## Appendix: Quick Reference

### HTTP Status Codes Cheat Sheet

| Code | Meaning | When to Use |
|------|---------|-------------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Client error (validation) |
| 401 | Unauthorized | Missing/invalid auth |
| 403 | Forbidden | Authenticated but not allowed |
| 404 | Not Found | Resource doesn't exist |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 503 | Service Unavailable | Server overloaded |

### gRPC Status Codes Cheat Sheet

| Code | Name | Meaning |
|------|------|---------|
| 0 | OK | Success |
| 1 | CANCELLED | Operation cancelled |
| 3 | INVALID_ARGUMENT | Client error |
| 4 | DEADLINE_EXCEEDED | Timeout |
| 5 | NOT_FOUND | Resource not found |
| 7 | PERMISSION_DENIED | Auth error |
| 13 | INTERNAL | Server error |
| 14 | UNAVAILABLE | Service down |
| 16 | UNAUTHENTICATED | Missing auth |

### Wireshark Filters Quick Reference

```
# HTTP/1.1
http.request.method == "POST"
http.response.code == 200

# HTTP/2
http2.streamid == 1
http2.type == 0  # DATA frame
http2.type == 1  # HEADERS frame

# gRPC
http2 and http2.header.content-type contains "application/grpc"
grpc.status_code != 0  # Errors only

# QUIC/HTTP/3
quic
quic.packet_type == 1  # Initial packets
quic.stream_id == 4    # Specific stream

# General
tcp.port == 443
udp.port == 443
ip.addr == 192.168.1.100
```

### Protocol Buffer Wire Types

| Wire Type | Used For | Binary Format |
|-----------|----------|---------------|
| 0 | Varint | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1 | 64-bit | fixed64, sfixed64, double |
| 2 | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 5 | 32-bit | fixed32, sfixed32, float |

### Load Testing Tools Comparison

| Tool | Best For | Language | Difficulty |
|------|----------|----------|------------|
| ab | Quick tests | Any | Easy |
| wrk | HTTP/1.1 & HTTP/2 | Lua scripting | Medium |
| k6 | Modern APIs | JavaScript | Medium |
| Artillery | Complex scenarios | YAML/JS | Medium |
| ghz | gRPC | Go | Easy |
| Gatling | Enterprise | Scala | Hard |

### Environment Setup Commands

```bash
# Node.js project setup
npm init -y
npm install express @apollo/server @grpc/grpc-js

# Python gRPC setup  
pip install grpcio grpcio-tools

# Generate gRPC code (Node.js)
grpc_tools_node_protoc \
  --js_out=import_style=commonjs:. \
  --grpc_out=grpc_js:. \
  --plugin=protoc-gen-grpc=$(which grpc_tools_node_protoc_plugin) \
  user.proto

# Generate gRPC-Web code
protoc -I=. user.proto \
  --js_out=import_style=commonjs:./generated \
  --grpc-web_out=import_style=typescript,mode=grpcwebtext:./generated

# Start Envoy for gRPC-Web
docker run -d -p 8080:8080 \
  -v $(pwd)/envoy.yaml:/etc/envoy/envoy.yaml \
  envoyproxy/envoy:v1.28-latest
  
# Install Wireshark
# macOS
brew install wireshark
# Ubuntu  
sudo apt-get install wireshark

# Install load testing tools
npm install -g artillery
go install github.com/bojand/ghz/cmd/ghz@latest
brew install k6  # or: https://k6.io/docs/get-started/installation/
```

---

**End of Guide**

*For updates, corrections, or contributions, visit the repository.*

*Last updated: December 2025*
