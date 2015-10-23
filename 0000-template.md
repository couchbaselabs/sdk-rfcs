# Meta

 - RFC Name: (unique feature name)
 - RFC ID: (pick the next id)
 - Start Date: (todays date, YYYY-MM-DD)
 - Owner: (your name)
 - Current Status: Draft/Review/Accepted

# Summary
One paragraph to explain the RFC

# Motivation
Why are we doing this? What use cases does it support? What is the expected outcome?

# General Design 
This part must cover the non-SDK specific API design. Concepts should be abstracted from an actual implementation, using some form of "pseudo" langue that describes the concept rather than the actual languate implementation.

This part also covers the unified behavior that the user sees. What errors does the user see? If user does X, what happens on Y?

# Language Specifics
In this section, the abstract design parts need to be broken down on the SDK level by each maintainer and signed off eventually.

| Generic | .NET         | Java   | NodeJS | Go | C | PHP | Python | Ruby |
| ------- | ------------ | ------ | ------ | -- | - | --- | ------ | ---- |
| foo()   | foo<dynamic> | Foo<D> | ...... | .. | . | ... | ...... | .... |

# Unresolved Questions
Put unresolved questions in here, this needs to be empty once the RFC reaches an accepted state.

# Signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| Java     | Michael N.     | 01.01.1960 |