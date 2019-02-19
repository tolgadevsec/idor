# Basic usage

The idor Node constructor accepts an unsigned int or UUID string identifier, and
a model type name that identifies the model class the identifier references.

The `.toString()` method returns a string that is safe to expose to the client side
(the "public id").

The `.valueOf()` method returns the original value (the "private id").

`Node.fromString()` can be used to create a new Node instance from a "public id"
string.

The `.typename` getter can be used to get the model class the identifier references.

```js
var Node = require("idor").default({ salt: "secret" });

new Node(1, "User").toString();
// '7Xb1vhHJpWvaDUXl+tRluA'

new Node(1, "Post").toString();
// '6PyZcANIhp1O/ydFDXHtsQ'

Node.fromString("7Xb1vhHJpWvaDUXl+tRluA").valueOf();
// 1

Node.fromString("7Xb1vhHJpWvaDUXl+tRluA").typename;
// 'User'
```

The constructor will also accept UUID values.

```js
new Node("123e4567-e89b-12d3-a456-426655440000", "User").toString();
// 'rfMD8jH0/sw4CFucgik+Yn8UStRYB/Mb7c8S770rpTc'

Node.fromString("rfMD8jH0/sw4CFucgik+Yn8UStRYB/Mb7c8S770rpTc").valueOf();
// '123e4567-e89b-12d3-a456-426655440000'

Node.fromString("rfMD8jH0/sw4CFucgik+Yn8UStRYB/Mb7c8S770rpTc").typename;
// 'User'
```

# Scoped usage

Scoping node IDs allows adding an additional layer of abstraction on identifiers
exposed to the client-side.

```js
new Node(1, "User", "private").toString();
// 'VgC9h2T/L6HgnQzCfftsBw'

Node.fromString("VgC9h2T/L6HgnQzCfftsBw").valueOf();
// null (wrong scope)

Node.fromString("VgC9h2T/L6HgnQzCfftsBw", "private").valueOf();
// 1

Node.fromString("VgC9h2T/L6HgnQzCfftsBw", "private").typename;
// 'User'
```

The default scope is 'public'.

```js
new Node(1, "User", "public").toString();
// '7Xb1vhHJpWvaDUXl+tRluA'
```

The `.scope` setter allows setting the scope after object initialisation

```js
const a = new Node(1, "User");
a.scope = "private";
a.toString();
// "VgC9h2T/L6HgnQzCfftsBw"

const b = Node.fromString("VgC9h2T/L6HgnQzCfftsBw");
b.scope = "private";
b.valueOf();
// 1
```

# Protect exposed identifiers

The application SHOULD set a unique salt to ensure identifiers cannot be computed
from exposed Node values, or generated externally.

```js
var Node = require("idor").default({ salt: "S3cr3t" });

new Node(1, "User").toString();
// 'qz5Rsa0czB+vsbwc0qjUcw'

new Node(1, "User", "private").toString();
// '0IWYZLpZFgfxPA8q3aNkBQ'

Node.fromString("7Xb1vhHJpWvaDUXl+tRluA").valueOf();
// null (wrong salt)

Node.fromString("qz5Rsa0czB+vsbwc0qjUcw").valueOf();
// 1
```

# Using with Apollo Server (GraphQL)

The `idor/graphql` module can be used to convert all ID types in an executable
schema to use indirect object references. This is mostly transparent to the
developer. Instead of seeing the private database ids, the obfuscated public ids
are visible to the GraphQL client.

Resolver functions will see idor Node objects instead of simple `id` values.
The `id.valueOf()` function will return the private database id.

```js
import { ApolloServer } from "apollo-server";
import { makeExecutableSchema } from "graphql-tools";
import idor from "idor/graphql";
import typedefs from "./schema.graphql";
import resolvers from "./graphql/resolvers";

const schema = idor(
  makeExecutableSchema({
    typeDefs,
    resolvers
  }),
  {
    salt: process.env.APP_KEY
  }
);
const context = request => {
  scope: contextSensitiveScope(request);
};
const server = new ApolloServer({ schema, context });
```

```js
// graphql/resolvers.js

const Query = {
  node(root, args) {
    const id = args.id.valueOf();
    const typename = args.id.typename;
    const Model = models.byTypename(typename);
    return Model.findById(id);
  },
  user(root, { id }) {
    return User.findById(id);
  }
};
```

# Using with Objection.js and Knex

[Objection.js](https://vincit.github.io/objection.js/) is an ORM for Node.js
that builds on [Knex](http://knexjs.org/).

`idor` Node objects have a `.toPostgres()` method used by the Knex query
builder. This means it's usually not necessary to manually call the `.valueOf()`
method on ID properties.

```js
import { Model } from "objection";
class User extends Model {
  static get tableName() {
    return "user";
  }
}

const id = Node.fromString("7Xb1vhHJpWvaDUXl+tRluA");
User.query()
  .findById(id)
  .then(user => {
    // user is an instance of the User class
  });
```

Validation of ID properties in Object.js models can be done using `jsonSchema`.

The`idor/jsonSchema` `uuid()` function returns a schema definition to validate
UUID strings, which also accept an idor Node object validating that the
typename property matches the correct type.

The `idor/jsonSchema` `uint()` function is similar, but for unsigned integer ids.

When doing this it is important to also provide the `jsonAttributes` getter
(which defines the properties that will be serialised as a JSON string when
being passed to the database). This is because the objection.js library will
try to auto-detect which attributes are JSON values by using the jsonSchema;
while idor Node objects are valid, they should _not_ be serialised.

```js
import { Model } from "objection";
import { uuid } from "idor/jsonSchema";

class User extends Model {
  static get tableName() {
    return "user";
  }
  static get jsonSchema() {
    return {
      type: "object",
      properties: {
        id: uuid("User")
      }
    };
  }
  static get jsonAttributes() {
    return [];
  }
}
```
