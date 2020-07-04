# ⚡ Using Apollo Server

## Quick notes on how to use apollo server for your GraphQL needs

### 📌 Server Setup

Below is an example of how to setup a basic `apollo server` for your graphql needs. It's better to have the server code in a seperate repo instead of colocating with the client repo.

Create a new server repo and do an `npm init` Then install the following `npm` packages:

```sh
npm install graphql apollo-server
```

Then create a new `index.js` file on the root with the following code:

```javascript
const { ApolloServer } = require("apollo-server");

// Schema (or) Type Definitions
const typeDefs = `
  type Query {
    hello: String
  }
`;

// Resolvers
const resolvers = {
  Query: {
    hello: () => "Hello World!"
  }
};

// Use port: 4000 to run the server
const PORT = process.env.PORT || 4000;

const server = new ApolloServer({
  typeDefs,
  resolvers
})
  .listen(PORT)
  .then(({ url }) => console.log(`🚀 running at ${url}`))
  .catch(console.error);
```

Now, on the terminal run `node index.js` to spin-up the server and navigate to the url `http://localhost:4000` to open-up the GraphiQL editor/UI.

For a better developer experience, you may setup `nodemon` to watch over your main `index.js` file and that takes care of all the automated server restarts alongside your server API changes.

---

### 📌 GraphQL Object Types and Lists

Using graphql object type, you can create list of objects and return them to the client. Here's an example:

```javascript
...
const typeDefs = `
  "Our new Order graphql object type"
  type Order {
    id: ID!,
    date: String!,
    product: String!,
    status: Status
  }

  "Enumerations can be added as well"
  enum Status {
    PROCESSING
    PENDING
    COMPLETE
  }

  "allOrders query returns a non-nullable"
  "list of Order objects"
  type Query {
    totalOrders: Int,
    allOrders: [Order!]!
  }
`;

let orders = [
  {
    id: "ord-123",
    date: "3/23/2020",
    product: "Mountain Bike",
    status: "PENDING"
  },
  {
    id: "ord-456",
    date: "4/8/2020",
    product: "Desktop Monitor",
    status: "COMPLETE"
  },
  {
    id: "ord-789",
    date: "1/12/2020",
    product: "Charcoal Grill",
    status: "PROCESSING"
  }
];

// `allOrders` query returns a list of `order` objects
const resolvers = {
  Query: {
    totalOrders: orders.length,
    allOrders: orders
  },
  Mutation: {...}
};
```

---

### 📌 Adding Mutations

Mutations allow you to change data on the server and return it back to the client. Deriving from the above example, here's how you can add a mutation to accept a new product order:

```javascript
// 'shortid' npm package lets us create unique
// short alpha-numeric ids.
import { generate } from "shortid";
...

// Add `mutations` under `typeDefs` using `type Mutation {}`
const typeDefs = `
  type Order {...}

  type Query {...}

  type Mutation {
    addOrder(
      date: String!
      product: String!
      status: Status
    ): Order
  }
`;

"'orders' object holds the list of current orders"
let orders = [...];

// Also, provide respective resolvers for the mutations.
const resolvers = {
  Query: {
    totalOrders: () => orders.length,
    allOrders: () => orders
  },

  "'addOrder' mutation takes in a 'parent' as the first"
  "argument and 'args' as the second argument. We can"
  "also destructure the props from args array."
  Mutation: {
    addOrder: (parent, { date, product, status }) => {
      "You can do input validations like below to ONLY allow the"
      "mutation to succeed if a valid product name was provided."
      if(product === "") {
        throw new Error("Product name cannot be blank!");
      }

      let newOrder = {
        id: generate(),
        date,
        product,
        status
      };

      "Add the new order to existing list of orders & return"
      "the new order back to the client."
      orders = [...orders, newOrder];
      return newOrder;
    }
  }
}
...
```

### 💡 Adding custom GraphQL scalar types to mutations

You can perform customizations on any of your input/incoming data within mutations by using a custom scalar type via the `GraphQLScalarType` class.

Below is an example of using a custom `Date scalar type` to transform various date inputs to the `addOrder` mutation described in the previous example. The specific changes are outlined below:

```javascript
const typeDefs = `
  "Our custom 'Date' scalar type"
  scalar Date

  type Order {...}

  type Mutation {
    "The date param is now using our custom non-nullable"
    "'Date' scalar type insted of 'String'"
    addOrder(
      date: Date!
      product: String!
      status: Status
    ): Order
  }

  enum Status {...}
  ...
`;

...

const resolvers = {
  Query: {...},
  Mutations: {
    addOrder...
  },
  // Our new `Date` scalar type with custom formatting to
  // transform input date values from the client.
  Date: new GraphQLScalarType({
    name: "Date",
    description: "A valid date value",
    serialize: value => value.substring(0, 10),
    parseValue: value => new Date(value).toISOString(),
    parseLiteral: literal => new Date(literal.value)
                                .toISOString()
  })
}
```

In the above example, the `serialize` function will always be called when we query this field. Likewise, the `parseValue` will be called when we pass data as variables within the mutation and `parseLiteral` when we pass data inline within the mutation.

Below is an example of how these types of mutations will look like in a query document:

```javascript
// 1. variable-based mutation
mutation (
  $date: Date!
  $product: String!
  $status: Status
) {
  addOrder (
    date: $date
    product: $product
    status: $status
  ) {
    id
    date
    product
    status
  }
}

// Query variables
{
  "date": "3/23/2020",
  "product": "Country Cookbook",
  "status": "PROCESSING"
}

// 2. literal-based mutation
mutation {
  addOrder (
    date: "2020-04-12",
    product: "Hiking Boots",
    status: "PENDING"
  ) {
    id
    date
    product
    status
  }
}
```

### 💡 Using GraphQL Input Types to wrap mutation arguments

To simplify things, we can wrap mutation arguments using graphql input types. Deriving from our previous examples, below is an example of how to use them in our `addOrder` mutation:

```javascript
const typeDefs = `
  scalar Date

  type Order {...}

  type Query {...}

  "Our new Graphql input type to wrap multiple arguments"
  "to the 'addOrder' mutation"
  input AddOrderInput {
    date: Date!
    product: String!
    status: Status
  }

  enum Status {...}

  type Mutation {
    "'addOrder' mutation now accepts a non-nullable 'AddOrderInput'"
    "Graphql input type instead of multiple arguments"
    addOrder(input: AddOrderInput!): Order
  }
`;

const resolvers = {
  Query: {...},

  Mutation: {
    // The arguments can now be destructured from the `input` key
    addOrder: (parent, { input: { date, product, status } }) => {
      ...
    }
  }
}
```

Now, in our query document we can call the mutation with the input type like so:

```javascript
// `addOrder` mutation with a custom input type.
mutation($input: AddOrderInput!) {
  addOrder(input: $input) {
    id
    date
    product
    status
  }
}

// Query variables for the mutation.
{
  "input": {
    "date": "3/22/2019",
    "product": "Computer Desk",
    "status": "PROCESSING"
  }
}
```

### 💡 Removing items from a list via Mutation

So far we have added new orders to our existing list of orders via the `addOrder` mutation. Similarly, we can also remove orders from our list via a `removeOrder` mutation. Below is an example of such a mutation.

Only the relevent pieces of code are shown below:

```javascript
// apollo server.js file

// ...
const typeDefs = `

  type Order {...}

  "Our custom payload returned to the client for"
  "when an order is removed"
  type RemoveOrderPayload {
    removed: Boolean
    totalBefore: Int
    totalAfter: Int
    removedOrder: Order
  }

  type Query {...}

  type Mutation {
    addOrder ...,
    // Our new 'removeOrder' mutation returns a custom
    // 'RemoveOrderPayload' object/shape
    removeOrder:(id: ID!): RemoveOrderPayload
  }
`;

const orders = [{ ... }];

const resolvers = {
  Query: {...},

  Mutation: {
    addOrder: ...,
    // Lookup the order from the client and remove it from the
    // current list of orders and return appropriate response.
    removeOrder: (parent, { id }) => {
      const totalBefore = orders.length;
      let removedOrder = orders.find(order => order.id === id);
      const removed = false;

      if(removedOrder) {
        orders = orders.filter(order => order.id !== id);
        removed = true;
      } else {
        throw new Error("Order not found or has already been removed!");
      }

      const totalAfter = orders.length;

      return {
        removed,
        totalBefore,
        totalAfter,
        removedOrder
      }
    }
  }
}
```

#### 👏 💥 🥁 🎉 🎊 🥳 Congratulations !! you made it through the end and that's a wrap on our quick reference guide on how to use Apollo Server for your GraphQL needs !!

#### 🏆 These notes are inspired by the amazing egghead tutorials on these topics by [Eve Porcello](https://egghead.io/instructors/eve-porcello) and [Alex Banks](https://egghead.io/instructors/alex-banks). They are a great resource for anyone starting with GraphQL and Apollo Server. Be sure to check them out on [egghead.io](https://egghead.io)

#### 🔥 If you like, you can also check out the sample [react-apollo app](https://github.com/jvikraman/react-apollo-graphql) that demonstrates the concepts discussed in this guide.
