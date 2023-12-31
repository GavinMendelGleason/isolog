# Isolog

This is an introduction to the Isolog data transducer system

Isolog is a strongly typed language with a complete latice over the
types.

The type-system is designed to make querying from a very broad set of
databases possible via query compilation.

This is done with the following concepts:

* Universally type
  - Allows us to describe unstructured data
* A *Refinement* calculus gives us the ability to migrate from untyped
  to typed smoothly
* Expansive primitive datatypes (arbitrary sized integers, floats etc.)
* Sums and products over datatypes
* Datatype constraints
  - This allows us to provide more specific primitive data types
    needed to represent diverse data sources
  - But also gives us a language to represent queries over resources

## The Problem Isolog solves

The data silo problem is as yet an unsolved problem. We have loads of
data, and the data is represented with multiple data models in
document stores, in relational databases and in graph databases. The
diversification of databases has given us more choice, but even *more*
data transformation headaches.

We need a way to model these different data sources such that this
data is not only discoverable, but accessible. And to do this we need
a tool powerful enough to describe a broad range of data.

Once we have appropriate descriptions we can faciliate data
transformation by providing a datalog query language rich enough to
faciliate almost arbitrary data transformation and discoverability
problems.

Instead of having one golden source, we can create a single metamodel
which allows our already existing data silos to be the golden source
of authority, or to carefully create new golden sources from amalgams
with ease.

## Primitive Types in Isolog

The primitve types are:

```
Bytes
Number
Integer
Rational
Float
Null
```

`Integer`, `Rational` and `Float` are indefinite precision, meaning
that more standard integer and float types can be seen as subsets, and
`Number` is the union of the three.

`String` is a predefined `utf-8` implementation of bytes, which
*morally* has the following definition:

```prolog
type String = { x : Bytes | is_utf8(x) }
```

This shows that string is a subset of bytes, i.e., `String < Bytes`.

We can form a subset of the integers which respresents u32s as:

```prolog
type U32 = { x : Integer | x >= 0 && x < 32**2 }
```

Or simply make a phone number field along the lines of:

```prolog
type PhoneNumber = { x : String | x =~ /[\d-]+/ }
```

## Algebraic Types

```
Pair
Sum
[_]
_!
```

iosolog also admits the above types as combinations of other
types.

```
type Coordinate = Float * Float
type StringOrFloat = String | Float
type FloatList = [Float]
type MaybeString = String!
```

In fact the `!` family is just syntactic sugar for `String | Null`

## Dictionaries Types

Dictionaries are key to modern programming, data transport and
modelling. Deep dictionaries with references (document graphs) are
capable of modelling most real-world data problems in a way that is
both computer and human frendly.

We therefore need a schema language which can model dictionaries and
for which it is relatively easy to "turn-up" the schema like a dial,
starting from untyped dictionaries, finally arriving at completely
typed descriptions (gradual typing).

The root type of the dictionary lattice is the `Dict`.

To create a dictionary with a specific field `name` which is required,
we can write:

A dictionary type starts from `Dict` and constrains the type, using a
number of operators. A dictionary type includes a number of fields and
types of those fields. For instance

```prolog
{ name : String, age: Integer }
```

We can create constraints by using the `<` and `>` subsumption, the
`|` union operator operators, or combining with disjoint union `[+]`.

For instance, we can say you either have a name field, or you have a
first name and last name field as follows:

```prolog
{ name : String } [+] { first_name : String, last_name : String }
```

```prolog
type Named = { x : Dict | x.name = y && y : String! }
```

This dictionary is *only* constrained by requiring an *optional*
`name` field, if it exists, to be a string (a not an arbitrary
type).

We can use a type subsumption operator to express this as well:

```prolog
type Named = { x : Dict |  x :> { name : String! } }
```

The `x :> T` says "The type of x subsumes T", or `x isa T && T > { name : String! }`

If we want the dictionary to be fully defined by the type spec, we can write:

```prolog
type Named = { x : Dict | x : { name : String! } }
```

And in fact, since the contraint is just a reiteration of the type, we can simply write:

```prolog
type Named = { name : String! }
```

This approach can be used to define all JSON dictionaries. Using
dictionaries and primitive types, we can write:

```prolog
type JSONLeaf = Null | String | Number
type JSON = { x : Dict | x[_] : (JSONLeaf | [JSON] | JSON) }
```

## Predicates

In isolog, predicates are named relations. A relation name can be
defined as connector to a database and with a name space.

```prolog
relation people < { x : Dict | x : { name : Utf8, id : Integer, age in Integer }}
                & Self.unique(id)
from { type: 'MySQL',
       host: 'localhost',
       name: 'HR',
       password: 'secret',
       user: 'admin',
       table: 'people' }
```

This says that the `People` relation is a subset of the dict records
above. Isolog will check to make sure that the table does in fact meet
this specification (i.e. that the type definition subsumes the table
definition).

## Queries

A query in isolog uses constraints and unification.

Using the above people relation, we can find all adaults in our
database with the following:

```prolog
people({ name : Name, id: Id, age: Age }), Age > 18.
```

Alternatively, since we don't care about the Name and the Id, we could write:

```prolog
people(Person), Person.age > 18.
```

This approach generalises over a large number of databases. For
instance, if we wanted to find a record in a mongo collection, we
could simply change the `from` definition we used.

Using this system we can easily federate queries over multiple
systems. The query compiler will try to arrange constraints such that
they are utilised in the query itself (by re-arranging constraints on
variables in the query order) but we can also utilise the constraint
information with indexes if these are known to exist.

To use an RDF database, such as TerminusDB, we can write something
along the lines of:

```prolog
relation event < { x : Dict | x : { name : Utf8, id : Integer, time : DateTime }}
                 & Self.unique(id)
from { type: 'TerminusDB',
       host: 'localhost',
       name: 'my_team/my_graph',
       password: 'secret',
       user: 'admin',
       class: 'Person' }
```

## Transactions

Transactions across federation are always a very difficult problem. We
can not guarantee that transactions will complete successfully.
