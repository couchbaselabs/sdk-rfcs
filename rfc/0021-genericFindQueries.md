# Meta

 - RFC Name: Generic Find Querying
 - RFC ID: 0021
 - Start Date: 2016-09-07
 - Owner: @simonbasle
 - Current Status: DRAFT

# Summary
Performing simple selects without writing any N1QL.

# Motivation
For simple querying cases, shortcircuiting N1QL altogether and having a simple
API can be valuable.

# General Design

## Query-by-Example
The goal of querying by example is to provide an example of what the user is
looking for in terms of criterias, and the SDK will return all matching
documents that match these criterias.

As there is no concept of an entity there, the criterias can simply be
expressed as a form of JSON object (so native JSON in Node, `JsonObject` in
Java, etc...).

Let's imagine that one wants to find every document that has a `lastname` field
equals to `Doe`, as well as an `age` field equals to `21`. This could be
achieved using query-by-example, eg. in Java:

```java
JsonObject example = JsonObject.create()
    .put("lastname", "Doe")
    .put("age", 21);

N1qlQueryResult result = bucket.findAllLike(example);
```

Which would translate, in the `Bucket` "foo", to the following N1QL:

```sql
SELECT * FROM `foo` WHERE lastname = 'Doe' AND age = 21;
```

## Matching on criterias other than `equals`
By default, all fields in the provided JSON example are matched against by
using the `=` N1QL operator. What if one wants a different criteria? For
instance, in our former example, what if I wanted all documents where the name
is "Doe" but the age is lesser than 21?

An alternative API for constructing the example could be proposed, that would
allow to both specify a value and an operator for each criteria.

The default simplified version of `findAllLike` would not provide the
operators, defaulting in effect each field to an `EQUALS` operator.

When done expressing the criterias, `findAllLike` would call on a `toN1ql()`
method to produce the full N1QL, for example:

```sql
SELECT * FROM `foo` WHERE lastname = "Doe" AND age <= 21;
```

The following operators would need to be supported:
 * `EQUALS`: implicit one used if a field from the example isn't found in the operators map.
 * `LIKE`
 * `CONTAINS`: translates to an `IN` keyword, inverted logic to be able to write something like "array CONTAINS aValue".
 * `IS_NULL`
 * `IS_MISSING`
 * `IS_VALUED`
 * `GREATER_THAN`
 * `GREATER_THAN_EQUALS`
 * `LESSER_THAN`
 * `LESSER_THAN_EQUALS`
 * `BETWEEN` (needs 2 values)
 * The negation of most of the above: `NOT_EQUALS`, `NOT_LIKE`, `NOT_CONTAINS`,
    `IS_NOT_NULL`, `IS_NOT_MISSING`, `IS_NOT_VALUED`, `NOT_BETWEEN`

### API alternative 1 (builder-friendly SDKs)
A *builder* like interface could be provided as a fluent API to specify both
fields, values and operators in one go:

```java
Example example = Example.of("lastname").not().equalTo("Doe")
    .and("age").greaterThan(20);

N1qlQueryResult result = bucket.findAllLike(example);
```

Would translate to this N1QL:

```sql
SELECT * FROM `foo` WHERE lastname != 'Doe' AND age > 20;
```

The builder methods in Java for each "positive" operator above would be:

```java
Criteria.of("a").equalTo("foo")
    .and("b").greaterThan(1)
    .and("c").greaterThanOrEqualTo(2)
    .and("d").lesserThan(4)
    .and("e").lesserThanOrEqualTo(3)
    .and("f").like("%toto%")
    .and("g").isNull()
    .and("h").isValued()
    .and("i").isMissing()
    .and("j").contains("bar")
    .and("k").between(100, 200);
```

### API alternative 2
For languages for which a builder API isn't practical, an alternative JSON
convention can be integrated.

Each operator mentioned above would be affected a string representation with a
`$` prefix, like `$gte` for `GREATER_THAN_EQUALS`. This string could then be
combined with the value in the JSON object passed to `findAllLike()`, as a sub
object with the operator as key and the value as value. For example in Node:

`findAllLike({ "lastName": { "$neq": "Doe" }, "age": { "$gt": 20 } })`

Which would also translate to this N1QL:

```sql
SELECT * FROM `foo` WHERE lastname != 'Doe' AND age > 20;
```

## Limitations
Anything other than a basic comparison operation as listed above is out of
scope for this RFC, and users must then craft their own N1QL statements (either
by hand or using any available DSL).

This goes for example for functions, array construction operators, CASE
constructs, ...

Criterias are for now only linked by a logical AND.

## Errors
N/A so far. The results are presented in the same form and with same error
handling conditions as with `Bucket#query`.

## Documentation
Probably needs its own section in the querying using the SDK pages.

# Language Specifics
TBD

# Questions
 - Should `or` be supported? (note this can increase the complexity for the builder API)

# Changelog
 - 2016-09-07: Initial draft publication
 - 2016-09-08: reworked the matchers/operators idea and builder, removed BETWEEN (added to open questions), added explicit list of "NOT" Operators.
 - 2016-09-15: Re-added between, expressed two alternatives to specify operators (JSON convention and builder API). CONTAINS should be used for the “is value x in the array field y?” case rather than a convenience method over LIKE.

# Signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
|          |                |            |