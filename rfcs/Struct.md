# `struct` and `structUnion` types

GraphQL currently has two types that are suitable for both input and output:
scalar and enum. These are both "leaf" types that _generally_ possess no
structure. This proposal is to add another type that's available on both input
and output, and yet is a composite type - one that has structure.

Before we think about _why_ we'd do this, lets take a look at the solution to
give some shared terminology.

## What would it look like?

There are two types that are close to solving this problem already:

- Input objects are already a structured input that handle many of our concerns,
  but they're only valid on input
- Scalars are already available on both input and output, and users can define
  their own

Originally, like with
[@oneOf](https://github.com/graphql/graphql-spec/pull/825), I considered this
proposal as a modification of one of these types. However, it feels like it
makes sense to introduce a new type, `struct`, that combines the advantages of
these two types. Additionally, to enable complex polymorphism, we introduce the
`structUnion` type.

**NOTE**: this syntax, these keywords, etc are not set in stone. Discussion is
very welcome!

### `struct`

A `struct` is composed of fields, each field has a type, and the type of a
`struct` field can be a `struct`, `structUnion`, scalar, enum, or a wrapping
type over any of these.

Importantly, when querying a `struct` you can never reach an object type, union
or interface from within a `struct` - the entire `struct` acts as a leaf.

Similarly, for input, a `struct` may never contain an input object.

Like for input objects, a `struct` may not contain an unbreakable cycle.

```graphql
struct Biography {
  title: String!
  socials: BiographySocials
  paragraphs: [Paragraph!]!
}
```

### `structUnion`

A `structUnion` is an abstract type representing that the value may be one of
many possible `struct`s.

```graphql
structUnion Paragraph =
  | TextParagraph
  | PullquoteParagraph
  | BlockquoteParagraph
  | TweetParagraph
  | GalleryParagraph
```

### Example schema

Here's an example schema where a user can customise their biography by composing
together paragraphs of different types. You could imagine that these paragraphs
were managed by an editor such as ProseMirror.

```graphql
struct Biography {
  title: String!
  socials: BiographySocials
  paragraphs: [Paragraph!]!
}

struct BiographySocials {
  github: String
  twitter: String
  linkedIn: String
  facebook: String
}

structUnion Paragraph =
  | TextParagraph
  | PullquoteParagraph
  | BlockquoteParagraph
  | TweetParagraph
  | GalleryParagraph

struct TextParagraph {
  text: String!
}
struct PullquoteParagraph {
  pullquote: String
  source: String
}
struct BlockquoteParagraph {
  paragraphs: [Paragraph!]!
  source: String
}
struct TweetParagraph {
  message: String!
  url: String!
}
struct GalleryParagraph {
  images: [Image!]!
}

struct Image {
  url: String!
  caption: String
}

type User {
  id: ID!
  username: String!
  bio: Biography!
}

type Query {
  user(id: ID!): User
}

type Mutation {
  setUserBio(userId: ID!, bio: Biography!): User
}
```

### Pure structured data

Structs are seen as raw data, the same on input as on output, and thus their
fields do not have resolvers, and do not accept arguments. They're just pure
structured data.

### Selection sets

Controversially, structs could be the first type in GraphQL that can be both a
leaf _and_ a non-leaf type - i.e. a selection set over them is optional. If you
do not provide a selection set then the entire object is returned. If you do
provide a selection set then those fields will be selected and returned.

Struct selection sets are similar to, but not the same as, regular selection
sets. In particular:

- struct fields are not `FIELD`s, i.e. they cannot have `FIELD` directives
  attached (suggest we call them `STRUCT_FIELD`)
- aliases are not allowed, otherwise the simple field merging (see below) cannot
  work

Both inline and named fragments are supported on `struct`s and `structUnion`s.

Field merging is very straightforward; for the example schema above, the
following query:

```graphql
{
  user(id: "1") {
    bio {
      title
    }
    bio {
      socials {
        twitter
      }
    }
  }
}
```

would be equivalent to:

```graphql
{
  user(id: "1") {
    bio {
      title
      socials {
        twitter
      }
    }
  }
}
```

(Note that the object with both title and socials satisfies both selection
sets.)

And similarly, the following:

```graphql
{
  user(id: "1") {
    ...A
    ...B
    ...C
  }
}

fragment A on User {
  bio {
    title
  }
}

fragment B on User {
  bio {
    socials {
      twitter
    }
  }
}

fragment C on User {
  # Select the entire field
  bio
}
```

would be equivalent to:

```graphql
{
  user(id: "1") {
    bio
  }
}
```

(Note that because we're selecting the entire `bio` field in `C`, all possible
subselections are satisfied.)

### `__typename`

`__typename` is available to query on all `struct`s (and thus on
`structUnion`s). This ensures that clients that automatically and silently add
`__typename` to every selection set will not break, and is also useful for
resolving the type of a `structUnion`.

On input of a `struct`, `__typename` is optional but recommended.

On input of a `structUnion`, `__typename` is required to determine the type of
the `struct` supplied.

If we were to make `__typename` required on all `struct` inputs then changing an
input type from `struct` to `structUnion` would be non-breaking; this is a nice
advantage, but is currently outweighed by the desire to make input pleasant for
users.

## Motivation

Motivation breaks into the following categories:

- providing a composite type that can be used for both input and output
- providing a composite type that's capable of polymorphism on input
- providing a type that treats a composite type as an atom (effectively allowing
  "wildcard" selection)
- providing a polymorphic type that's symmetric across input and output

### Why introduce a structured input/output equivalent type?

There has long been a desire in the community for allowing custom scalars to
have a defined structure and yet still be treated as "atomic" (i.e. pull down
the entire object, not just a selection). In some schemas this has been solved
by using a `JSON` scalar that represents the composite type as a JSON encoded
string, or even the JSON object directly; but such scalars do not define
structure explicitly.

One example of where this is useful is with traditional RESTful APIs which use
the `PUT` (rather than `PATCH`) mechanic - i.e. they replace the entire object
rather than "patching" it. To allow us to do this in a future compatible way
(allowing for the object to gain more properties over time) we must be careful
not to accidentally drop properties that we do not understand, so we should pull
down the entire object, modify the parts that we want to (leaving parts we don't
understand alone) and then send the entire (modified) object back to the server.

Another example is the
[huge thread on adding support for wildcards](https://github.com/graphql/graphql-spec/issues/942).
In general GraphQL, wildcards have a lot of questions:

- Which fields do we select?
- What do we do for fields with arguments?
- What do we do about deprecated fields?
- Do we automatically add subselections to non-leaf fields?
  - How deep do we go?
  - What happens if there's an infinite loop?
- Etc.

And they also impede the "versionless schema" desire of GraphQL - if a client
uses wildcards then when new fields are added the client will start receiving
these too, sending them data that they may not understand and causing versioning
problems.

However, if we constraint the problem space down to that of pure structured data
(treating it like an atom - like a `scalar` is currently) then many of these
concerns lessen.

## Should `struct` replace input objects?

Maybe... They do seem to solve many of the same problems. However one major
difference is intent; because `struct` is intended to be used for input/output
it's intended that you pull it down, modify it as need be, and then send it back
again. As such, adding a non-nullable field to a struct used in this way may not
be a breaking change - existing clients should automatically receive the
additional fields, and send them back unmodified.

The semantics of `struct` and input objects are very similar, so it could be
that we repurpose input objects for struct usage; this will need further
thought, discussion and investigation. Not least because the term `input` would
make it confusing ;)
