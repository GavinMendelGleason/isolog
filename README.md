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

## Types in Isolog

The primitve types are:

```
Bytes
Integer
Rational
Float
Null
Dictionary
Pair
Sum
Array
```

`Integer`, `Rational` and `Float` are indefinite precision, meaning
that more standard integer and float types can be seen as subsets.

`String` is a predefined `utf-8` implementation of bytes, which
*morally* has the following definition:

```prolog
ptype String = { x : Bytes | is_utf8(x) }
```

We can form a subset of the integers which respresents u32s as:

```prolog
ptype U32 = { x : Integer | x < 32**2 && self >= 0 }
```

Or simply make a phone number field along the lines of:

```prolog
ptype PhoneNumber = { x : String | x =~ /[\d-]+/ }
```

```prolog
type Person < Top {
  name: String!
}

type Anything {
  ...
}

```
