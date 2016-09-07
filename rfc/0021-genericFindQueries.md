# Meta

 - RFC Name: Generic Find Querying
 - RFC ID: 0021
 - Start Date: 2016-09-07
 - Owner: @simonbasle
 - Current Status: DRAFT

# Summary
Performing simple selects without writing any N1QL.

# Motivation
For simple querying cases, shortcircuiting N1QL altogether and having a simple API can be valuable.

# General Design

## Query-by-Example
The goal of querying by example is to provide an example of what the user is looking for in terms of criterias, and the SDK will return all matching documents that match these criterias.

As there is no concept of an entity there, the criterias can simply be expressed as a form of JSON object (so native JSON in Node, `JsonObject` in Java, etc...).

Let's imagine that one wants to find every document that has a `lastname` field equals to `Doe`, as well as an `age` field equals to `21`. This could be achieved using query-by-example, eg. in Java:

```java
JsonObject example = JsonObject.create()
    .put("lastname", "Doe")
    .put("age", 21);

N1qlQueryResult result = bucket.findAll(example);
```

Which would translate, in the `Bucket` "foo", to the following N1QL:

```sql
SELECT * FROM `foo` WHERE lastname = 'Doe' AND age = 21;
```

## Matching on criterias other than `equals`
By default, all fields in the provided JSON example are matched against by using the `=` N1QL operator. What if one wants a different criteria? For instance, in our former example, what if I wanted all documents where the name is "Doe" but the age is lesser than 21?

The `findAll` method could in fact be based on a map between JSON dict keys and Matchers. Each Matcher would wrap the operator and the value(s) to match against.

The default simplified version of `findAll` would convert the provided example into such a map where all matchers are simple `EQUALS` matchers.

```java
JsonObject example = JsonObject.create()
    .put("lastname", "Doe")
    .put("age", 21);
Map<String, Operator> matchers = Collections.singletonMap("age", Operator.LESS_THAN);

N1qlQueryResult result = bucket.findAll(example, matchers);
```

In the background, both `example` and `matchers` would be combined to produce the following matchers specifications:
 - `Matcher{field="lastname", operator=EQUALS, value="Doe"}`
 - `Matcher{field="age", operator=LESS_THAN, value=21}`

Which would in turn be convertible to N1QL:

```sql
SELECT * FROM `foo` WHERE lastname = 'Doe' AND age < 21;
```

The following operators would need to be supported:
 * `EQUALS`: implicit one used if a field from the example isn't found in the matchers map.
 * `LIKE`
 * `CONTAINS`: similar to `LIKE`, but automatically convert the example value to string and encloses it with `%` if not present (ie. `nickname CONTAINS 2` will give `WHERE nickname LIKE '%2%'`).
 * `IS_NULL`
 * `IS_MISSING`
 * `GREATER_THAN`
 * `GREATER_THAN_EQUALS`
 * `LESSER_THAN`
 * `LESSER_THAN_EQUALS`
 * `VALUED`
 * `BETWEEN`: matches against 2 values, the second one being inclusive in the range (BETWEEN N1QL operator).
 * The negation of all the above, prefixed by `NOT_`


As a third alternative, a *builder* like interface could be provided as a fluent API for directly creating all these Matchers in one go:

```java
Example example = Example.of("lastname", not(equals("Doe")))
    .and("age", between(10, 20));

N1qlQueryResult result = bucket.findAll(example);
```

Would translate to these matcher specifications:
 - `Matcher{field="lastname", operator=NOT_EQUALS, value="Doe"}`
 - `Matcher{field="age", operator=BETWEEN, value=[10, 20]}`

That would result in N1QL:

```sql
SELECT * FROM `foo` WHERE lastname != 'Doe' AND age BETWEEN 10 AND 20;
```

## Limitations
Anything other than a basic comparison operation as listed above is out of scope for this RFC, and users must then craft their own N1QL statements (either by hand or using any available DSL). This goes for instance for functions, array construction operators, CASE constructs...

## Errors
N/A so far. The results are presented in the same form and with same error handling conditions as with `Bucket#query`.

## Documentation
Probably needs its own section in the querying using the SDK pages.

# Language Specifics
TBD

# Questions
TBD

# Changelog
 - 2016-09-07: Initial draft publication

# Signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
|          |                |            |