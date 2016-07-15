*MetaMeta:* Usually RFCs start off as GDrive Docs and then become Markdown as they're completed. There is a [handy GDrive Template too](https://docs.google.com/document/d/1z3doehquTjXDL6mimQH3Eyp7pv80r0tsPebvtx59j9A/edit#heading=h.pja6dcwixhvq).

# Meta

 - RFC Name: [TBD: unique feature name]
 - RFC ID: [TBD: pick the next id]
 - Start Date: TBD: todays date, YYYY-MM-DD]
 - Owner: [TBD: your name]
 - Current Status: TBD: DRAFT -> REVIEW -> ACCEPTED]

# Summary
[TBD: A short paragraph summary of the SDK-RFC.  Something appropriate for putting in a status dashboard.]

# Motivation
[TBD: Why are we doing this? What use cases does it support? What is the expected outcome?]

# General Design 
[TBD: This part must cover the non-SDK specific API design. Concepts should be abstracted from an actual implementation, using some form of "pseudo" language that describes the concept rather than the actual language implementation.
This part also covers the unified behavior that the user sees. What errors does the user see? If user does X, what happens on Y?
What is the scope?  What are the main components?  How do they interact?  What new interfaces are created?  How are they presented in each of the platforms in scope?  How will you know when you are done?]

## Errors
[TBD: What new error handling needs to be considered]

## Documentation
[TBD: What documentation artifacts need to be created or changed in support of this RFC]

# Language Specifics
In this section, the abstract design parts need to be broken down on the SDK level by each maintainer and signed off eventually.

| Generic | .NET         | Java   | NodeJS | Go | C | PHP | Python | Ruby |
| ------- | ------------ | ------ | ------ | -- | - | --- | ------ | ---- |
| foo()   | foo<dynamic> | Foo<D> | ...... | .. | . | ... | ...... | .... |

# Questions
[TBD: A running log of larger questions, unresolved at first and later kept in place with their answers if the owner considers it useful to keep the background around.]

# Changelog
[TBD: YYYY-MM-DD
Initial draft publication
]

# Signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| Java     | Michael N.     | 01.01.1960 |
