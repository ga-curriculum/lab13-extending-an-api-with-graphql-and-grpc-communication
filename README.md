## Lab: Extending an API with GraphQL and gRPC Communication

**Prerequisites:**

* Node.js installed
* Basic knowledge of REST APIs and JavaScript
* Basic understanding of GraphQL and gRPC concepts
* Existing REST API project setup (students should have a simple API with one or two resources, e.g., `Users` and `Products`)

### **Lab Objectives**

By the end of this lab, you will be able to:

1. Add a GraphQL layer on top of an existing REST API.
2. Define GraphQL schemas, queries, and resolvers.
3. Implement a simple gRPC service for communication between microservices.
4. Connect a GraphQL query to a gRPC service to fetch data.

---

### **Lab Setup**

1. **Clone Starter Project**

   ```bash
   git clone https://github.com/example/rest-api-starter.git
   cd rest-api-starter
   npm install
   npm start
   ```

   * The project includes two REST endpoints:

     * `GET /users` → returns a list of users
     * `GET /products` → returns a list of products

2. **Install Required Dependencies**

   ```bash
   npm install graphql express-graphql @grpc/grpc-js @grpc/proto-loader
   ```

---

### **Part 1: Adding a GraphQL Layer**

**Step 1: Create a GraphQL Schema**

* Create a file `schema.js`:

```javascript
const { GraphQLObjectType, GraphQLSchema, GraphQLString, GraphQLList } = require('graphql');

// Sample in-memory data (can later connect to REST API or gRPC)
const users = [
  { id: '1', name: 'Alice' },
  { id: '2', name: 'Bob' },
];

const UserType = new GraphQLObjectType({
  name: 'User',
  fields: {
    id: { type: GraphQLString },
    name: { type: GraphQLString },
  },
});

const RootQuery = new GraphQLObjectType({
  name: 'RootQueryType',
  fields: {
    users: {
      type: new GraphQLList(UserType),
      resolve() {
        return users;
      },
    },
  },
});

module.exports = new GraphQLSchema({
  query: RootQuery,
});
```

**Step 2: Add GraphQL Middleware to Express**

* In `index.js` or your main server file:

```javascript
const express = require('express');
const { graphqlHTTP } = require('express-graphql');
const schema = require('./schema');

const app = express();

app.use('/graphql', graphqlHTTP({
  schema,
  graphiql: true,
}));

app.listen(4000, () => console.log('Server running on http://localhost:4000/graphql'));
```

**Step 3: Test GraphQL Queries**

* Open `http://localhost:4000/graphql` in the browser and try:

```graphql
{
  users {
    id
    name
  }
}
```

---

### **Part 2: Adding gRPC Communication**

**Step 1: Define a gRPC Service**

* Create a file `user.proto`:

```proto
syntax = "proto3";

package userservice;

service UserService {
  rpc GetUserById (UserRequest) returns (UserResponse);
}

message UserRequest {
  string id = 1;
}

message UserResponse {
  string id = 1;
  string name = 2;
}
```

**Step 2: Implement gRPC Server**

* Create `grpcServer.js`:

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const packageDef = protoLoader.loadSync('user.proto', {});
const grpcObj = grpc.loadPackageDefinition(packageDef);
const userService = grpcObj.userservice;

const users = [
  { id: '1', name: 'Alice' },
  { id: '2', name: 'Bob' },
];

function getUserById(call, callback) {
  const user = users.find(u => u.id === call.request.id);
  callback(null, user || {});
}

const server = new grpc.Server();
server.addService(userService.UserService.service, { GetUserById: getUserById });
server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
  server.start();
  console.log('gRPC server running on 0.0.0.0:50051');
});
```

**Step 3: Connect GraphQL to gRPC**

* Update `schema.js`:

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDef = protoLoader.loadSync('user.proto', {});
const grpcObj = grpc.loadPackageDefinition(packageDef);
const client = new grpcObj.userservice.UserService('localhost:50051', grpc.credentials.createInsecure());

const RootQuery = new GraphQLObjectType({
  name: 'RootQueryType',
  fields: {
    user: {
      type: UserType,
      args: { id: { type: GraphQLString } },
      resolve(parent, args) {
        return new Promise((resolve, reject) => {
          client.GetUserById({ id: args.id }, (err, response) => {
            if (err) reject(err);
            else resolve(response);
          });
        });
      },
    },
  },
});
```

**Step 4: Test GraphQL Query with gRPC**

```graphql
{
  user(id: "1") {
    id
    name
  }
}
```

---

### **Part 3: Lab Wrap-up**

* Ensure students can query users via GraphQL using both in-memory and gRPC-backed resolvers.
* Quick recap questions:

  * What advantages does GraphQL provide over REST?
  * How does gRPC improve microservice communication?

---

### **Lab Deliverables**

1. GraphQL endpoint running at `/graphql` with queries for `users` and `user(id: String)`.
2. gRPC server running at `localhost:50051` serving `UserService`.
3. A working GraphQL query that fetches data via gRPC.

