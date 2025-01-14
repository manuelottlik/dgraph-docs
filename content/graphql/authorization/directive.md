+++
title = "The @auth Directive"
description = "Given an authentication mechanism and a signed JSON Web Token (JWT), the @auth directive tells Dgraph how to apply authorization."
weight = 2
[menu.main]
    parent = "authorization"
+++

The `@auth` directive tells Dgraph how to apply authorization. You can use it
to define authorization rules for most types (except for `union` and `@remote`
types). It lets you control which users can run which queries - as well as
which users can add, update, and delete data using mutations. 

Additionally, you can use this directive with the [`@secret`](/graphql/schema/types#password-type)
directive; and, if you specify a `password` authorization rule, Dgraph will use
it to authorize the `check<Type>Password` query.

{{% notice "note" %}}
The [Union type](/graphql/schema/types#union-type) does not support the `@auth` directive. 
{{% /notice %}}

You create authorization rules using the `@auth` directive, and those rules are
expressed in exactly the same syntax as GraphQL queries. Why? Because the
authorization you add to your app is about the graph of your application, so
graph rules make sense.  It's also a syntax that you already know about, and you
get syntax help from GraphQL tools in writing such rules. And these are exactly
the kinds of rules Dgraph already knows how to evaluate.

## Authorization rules

A valid type and rule looks like the following:

```graphql
type Todo @auth(
    query: { rule: """
        query ($USER: String!) { 
            queryTodo(filter: { owner: { eq: $USER } } ) { 
                id 
            } 
        }"""
    }
){
    id: ID!
    text: String! @search(by: [term])
    owner: String! @search(by: [hash])
}
```

{{% notice "note" %}}
To use the `@auth` directive, you must configure the authentication method used
by Dgraph in the last line of your schema with a `Dgraph.Authorization` object,
as described in the [Authorization Overview](/graphql/authorization/authorization-overview).
{{% /notice %}}

Here we define a type `Todo`, that has an `id`, the `text` of the todo and the username of the `owner` of the todo.  What todos can a user query?  Any `Todo` that the `query` rule would also return.

The `query` rule in this case expects the JWT to contain a claim `"USER": "..."` giving the username of the logged in user, and says: you can query any todo that has your username as the owner.

In this example we use the `queryTodo` query that will be auto generated after uploading this schema. When using a query in a rule, you can only use the `queryTypeName` query. Where `TypeName` matches the name of the type, where the `@auth` directive is attached. In other words, we could not have used the `getTodo` query in our rule above to query by id only.

This rule is applied automatically at query time.  For example, the query

```graphql
query {
    queryTodo {
        id
        text
    }
}
```

will return only the todo's where `owner` equals `amit`, when Amit is logged in and only the todos owned by `nancy` when she's logged into your app.

Similarly,

```graphql
query {
    queryTodo(filter: { text: { anyofterms: "graphql"}}, first: 10, order: { asc: text }) {
        id
        text
    }
}
```

will return the first ten todos, ordered ascending by title of the user that made the query.

This means your frontend doesn't need to be sensitive to the auth rules.  Your app can simply query for the todos and that query behaves properly depending on who's logged in.

In general, an auth rule should select a field that's expected to exist at the inner most field, often that's the `ID` or `@id` field.  Auth rules are run in a mode that requires all fields in the rule to find a value in order to succeed.  

## `@auth` on Interfaces

The `@auth` directive works just like it does for types, and provides authorization to perform `query`, `update`, and `delete` on interfaces.

### Implementing types

The rules provided inside the `@auth` directive on an interface will also be applied as an `AND` rule to those on the implementing types.

{{% notice "tip" %}}
A type inherits the `@auth` rules of all the implemented interfaces. The final authorization rule is an `AND` of the type's `@auth` rule and of all the implemented interfaces.
{{% /notice %}}

In the following example, the `Question` and `Answer` types will automatically inherit the auth rules of the `Post` type. 
This means that a user can only query a subset of questions and answers that are accessible through the `queryPost` query. 
Dgraph will disallow situations where a user can query more posts through `queryAnswer` or `queryQuestion` than they can through `queryPost`.

```graphql
type Author {
  id: ID!
  name: String! @search(by: [hash])
  posts: [Post] @hasInverse(field: author)
}

interface Post @auth(
    # The JWT should contain a claim "USER", which should be the ID of the logged in author.
    # This query rule would ensure that the logged-in author sees only their posts.
    query: { rule: """
        query ($USER: ID!) { 
            queryPost {
              author(filter: { id: [$USER] }) {
                id
              }
            } 
        }"""
    }
){
  id: ID!
  text: String @search(by: [fulltext])
  datePublished: DateTime @search
  author: Author! 
}

type Question implements Post @auth(
    # The JWT should contain a boolean claim "ANSWERED".
    # This query rule would ensure that only the questions that have the answered field matching 
    # the ANSWERED claim are returned in response.
    query: { rule: """
        query ($ANSWERED: Boolean!) { 
            queryQuestion(filter: { answered: $ANSWERED } ) { 
                id 
            } 
        }"""
    }
){
  answered: Boolean
}

type Answer implements Post @auth(
    # The JWT should contain a boolean claim "USEFUL".
    # This query rule would ensure that only the answers that have the markedUseful field matching
    # the USEFUL claim are returned in response.
    query: { rule: """
        query ($USEFUL:Boolean!) { 
            queryAnswer(filter: { markedUseful: $USEFUL } ) { 
                id 
            } 
        }"""
    }
){
  markedUseful: Boolean
}
```

If the `Question` type implemented more interfaces, then the rules for those would also be added in an `AND` condition to the `Question` type's authorization rules.

### Interfaces

When it comes to applying `@auth` rules on interfaces themselves, Dgraph will do a `union` query where it queries all the implementing types, and apply the authorization rules on them. 
The final query will be an `OR` query joining the results from all the implementing types.

### Mutations

Mutations on an interface works in the same manner. For example, in case of a `delete` mutation on an interface, it will be broken down into the implementing type's `delete` mutation. The nodes that satisfy the `@auth` rules of the corresponding implementing types and the interface will get deleted.

## Graph traversal in auth rules

Often authorization depends not on the object being queried, but on the connections in the graph that object has or doesn't have.
Because the auth rules are graph queries, they can express very powerful graph search and traversal.

For a simple todo app, it's more likely that you'll have types like this:

```graphql
type User {
    username: String! @id
    todos: [Todo]
}

type Todo {
    id: ID!
    text: String!
    owner: User
}
```

This means your auth rule for todos will depend not on a value in the todo, but on checking which owner it's linked to.
This means our auth rule must make a step further into the graph to check who the owner is.

So, you will have to modify the above schema like this:

```graphql
type User {
	username: String! @id
	todos: [Todo]
}

type Todo @auth(
    query: { rule: """
        query ($USER: String!) {
            queryTodo {
                owner(filter: { username: { eq: $USER } } ) {
                    username
                }
            }
        }"""
    }
){
	id: ID!
	text: String!
	owner: User
}
```

You can express a lot with these kinds of graph traversals.  For example, multitenancy rules can express that you can only see an object if it's linked (through what ever graph search you define) to the organization you were authenticated from.  That means your app can split data per customer easily.

You can also express rules that can be administered by the app itself.  You might define type `Role` and enum `Privileges` that can have values like `VIEW`, `ADD`, etc. and state in your auth rules that a user needs to have a role with particular privileges to query/add/update/delete and those roles can then be allocated inside the app.  For example, in an app about project management, when a project is created the admin can decide which users have view or edit permission, etc.

## Role Based Access Control

As well as rules that relate a user's claims to a graph traversal, role based access control rules are also possible.  These rules relate a claim in the JWT to a known value.  

For example, perhaps only someone logged in with the `ADMIN` role is allowed to delete users.  For that, we might expect the JWT to contain a claim `"ROLE": "ADMIN"`, and can thus express a rule that only allows users with the `ADMIN` claim to delete.

```graphql
type User @auth(
    delete: { rule:  "{$ROLE: { eq: \"ADMIN\" } }"}
) { 
    username: String! @id
    todos: [Todo]
}
```

Not all claims need to be present in all JWTs.  For example, if the `ROLE` claim isn't present in a JWT, any rule that relies on `ROLE` simply evaluates to false.  As well as simplifying your JWTs (e.g. not all users need a role if it doesn't make sense to do so), this means you can also simply disallow some queries and mutations.  If you know that your JWTs never contain the claim `DENIED`, then a rule such as

```graphql
type User @auth(
    delete: { rule:  "{$DENIED: { eq: \"DENIED\" } }"}
) { 
    ...
}
```

can never be true and this would prevent users ever being deleted.

## and, or & not 

Rules can be combined with the logical connectives and, or and not, so a permission can be a mixture of graph traversals and role based rules.

In the todo app, you can express, for example, that you can delete a `Todo` if you are the author, or are the site admin.

```graphql
type Todo @auth(
    delete: { or: [ 
        { rule:  "query ($USER: String!) { ... }" }, # you are the author graph query
        { rule:  "{$ROLE: { eq: \"ADMIN\" } }" }
    ]}
)
```

## Public Data

Many apps have data that can be accessed by anyone, logged in or not.  That also works nicely with Dgraph auth rules.  

For example, in Twitter, StackOverflow, etc. you can see authors and posts without being signed it - but you'd need to be signed in to add a post.  With Dgraph auth rules, if a type doesn't have, for example, a `query` auth rule or the auth rule doesn't depend on a JWT value, then the data can be accessed without a signed JWT.

For example, the todo app might allow anyone, logged in or not, to view any author, but not make any mutations unless logged in as the author or an admin.  That would be achieved by rules like the following.

```graphql
type User @auth(
    # no query rule
    add: { rule:  "{$ROLE: { eq: \"ADMIN\" } }" },
    update: ...
    delete: ...
) {
    username: String! @id
    todos: [Todo]
}
```

Maybe some todos can be marked as public and users you aren't logged in can see those.

```graphql
type Todo @auth(
    query: { or: [
        # you are the author 
        { rule: ... },
        # or, the todo is marked as public
        { rule: """query { 
            queryTodo(filter: { isPublic: { eq: true } } ) { 
                id 
            } 
        }"""}
    ]}
) { 
    ...
    isPublic: Boolean
}

```

Because the rule doesn't depend on a JWT value, it can be successfully evaluated for users who aren't logged in.

Ensuring that requests are from an authenticated JWT, and no further restrictions, can be done by arranging the JWT to contain a value like `"isAuthenticated": "true"`.  For example,


```graphql
type User @auth(
    query: { rule:  "{$isAuthenticated: { eq: \"true\" } }" },
) {
    username: String! @id
    todos: [Todo]
}
```

specifies that only authenticated users can query other users.

---
