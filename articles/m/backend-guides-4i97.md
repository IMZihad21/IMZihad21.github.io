# Essential Concepts for JavaScript Developers in Back-End Development

- Canonical URL: https://imzihad21.github.io/articles/a/backend-guides-4i97/
- Source URL: https://dev.to/imzihad21/backend-guides-4i97
- Web View: https://imzihad21.github.io/articles/a/backend-guides-4i97/
- Published: 2021-12-23T16:53:07.000Z
- Modified: 2021-12-23T16:53:07.000Z
- Reading time: 3 minutes
- Tags: crud, jwt, api, database

If you already know JavaScript from frontend work, backend development is the next strong move. You will work more with data, security, APIs, and infrastructure decisions.

This guide keeps the fundamentals practical so you can build robust server-side applications without overcomplicating the stack.

### Why It Matters

- Back-end skills help you build full features, not only UI screens.
- You can design secure APIs and control your data layer directly.
- You make better architecture decisions across the full stack.
- You reduce handoff friction between frontend and backend teams.

### Core Concepts

#### 1. CRUD Operations

CRUD means the four core data operations in most applications:

- Create: add new records.
- Read: fetch existing records.
- Update: modify records.
- Delete: remove records.

These operations are the backbone of REST APIs for resources like users, products, and orders.

#### 2. JWT for Authentication

JSON Web Token (JWT) is commonly used for stateless authentication and authorization.

- After login, the server issues a signed token.
- The client sends that token on future requests.
- The server verifies the signature before allowing protected actions.

JWT has three parts: header, payload, and signature.

#### 3. NoSQL Databases

NoSQL databases are flexible and horizontally scalable.

- Document stores: MongoDB
- Key-value stores: Redis
- Column stores: Cassandra
- Graph databases: Neo4j

For JavaScript developers, document databases feel natural because data structure is JSON-like.

#### 4. Relational Databases and SQL

Relational databases store data in tables with strict structure.

- Primary keys uniquely identify rows.
- Foreign keys define relationships.
- SQL supports filtering, joins, sorting, and aggregations.

Use relational databases when data integrity and complex relationships are critical.

#### 5. Aggregation for Analytics

Aggregation summarizes data for reports and dashboards.

- SQL functions like `COUNT`, `SUM`, and `AVG` are core tools.
- MongoDB aggregation pipelines allow advanced transformation flows.

When your app needs insights, aggregation becomes mandatory, not optional.

#### 6. Express.js for APIs

Express.js is a lightweight Node.js framework for building APIs.

- Routing defines endpoints like `/users` and `/orders`.
- Middleware handles auth, validation, and request processing.
- Modularity lets you scale features without rewriting the core server.

### Practical Example

Basic Express API with CRUD routes and JWT-protected endpoint:

```javascript
import express from "express";
import jwt from "jsonwebtoken";

const app = express();
app.use(express.json());

const jwtSecret = process.env.JWT_SECRET || "dev_secret_change_me";

const users = [];
let nextUserId = 1;

function authenticateToken(req, res, next) {
  const authHeader = req.headers.authorization;
  const token = authHeader?.startsWith("Bearer ") ? authHeader.slice(7) : null;

  if (!token) {
    return res.status(401).json({ message: "Missing token" });
  }

  try {
    const payload = jwt.verify(token, jwtSecret);
    req.user = payload;
    return next();
  } catch {
    return res.status(401).json({ message: "Invalid token" });
  }
}

app.post("/auth/login", (req, res) => {
  const { username } = req.body;

  if (!username) {
    return res.status(400).json({ message: "Username is required" });
  }

  const token = jwt.sign({ username }, jwtSecret, { expiresIn: "1h" });
  return res.json({ token });
});

app.post("/users", authenticateToken, (req, res) => {
  const { name, email } = req.body;

  if (!name || !email) {
    return res.status(400).json({ message: "Name and email are required" });
  }

  const user = { id: nextUserId, name, email };
  nextUserId += 1;
  users.push(user);

  return res.status(201).json(user);
});

app.get("/users", authenticateToken, (req, res) => {
  return res.json(users);
});

app.put("/users/:id", authenticateToken, (req, res) => {
  const userId = Number(req.params.id);
  const user = users.find((item) => item.id === userId);

  if (!user) {
    return res.status(404).json({ message: "User not found" });
  }

  const { name, email } = req.body;
  user.name = name ?? user.name;
  user.email = email ?? user.email;

  return res.json(user);
});

app.delete("/users/:id", authenticateToken, (req, res) => {
  const userId = Number(req.params.id);
  const userIndex = users.findIndex((item) => item.id === userId);

  if (userIndex === -1) {
    return res.status(404).json({ message: "User not found" });
  }

  users.splice(userIndex, 1);
  return res.status(204).send();
});

app.listen(3000, () => {
  console.log("API running on http://localhost:3000");
});
```

This is a simple in-memory setup. In production, replace the array with a real database before your users become historical artifacts.

### Common Mistakes

- Building routes without clear resource naming and HTTP semantics.
- Storing JWT secrets in source code instead of environment variables.
- Using NoSQL by default without checking relational requirements.
- Skipping input validation and returning inconsistent error responses.
- Writing analytics queries without indexes and expecting magic speed.

### Quick Recap

- CRUD is the operational base for API data management.
- JWT supports stateless authentication when implemented correctly.
- NoSQL and SQL solve different classes of problems.
- Aggregation powers reporting and business insights.
- Express.js provides a practical framework for building Node.js APIs.

### Next Steps

1. Replace in-memory storage with PostgreSQL or MongoDB.
2. Add request validation using a schema library.
3. Add refresh-token strategy for stronger auth flows.
4. Create one analytics endpoint using SQL or MongoDB aggregation.