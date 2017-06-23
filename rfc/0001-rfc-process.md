# Meta

 - RFC Name: The RFC Process
 - RFC ID: 0001
 - Start Date: 2015-10-23
 - Owner: Michael Nitschinger
 - Current Status: Accepted

# Summary
This RFC describes the newly established RFC process.

# Motivation
Since the inception of the "SDK", we struggled with inconsistencies between the different language implementations. Many different reasons are contributing to it (legacy codebases, customer escalations, forgetting to communicate,...), but in the end there are two noticeable problems to the user:

 - Syntactical API inconsistencies
 - Semantical behavior differences in cluster communication

Both lead to user dissatisfaction. The first one is often caught during development, the second one very often hits the user in production. In order to provide a better user experience, we should strive to improve our communication and synchronization activities.

Note that this is different from PRDs that PM creates. These RFCs should get into the weeds and nitty gritty details of technical implementation and behavior.

The RFC process is designed to be lightweight, collaborative and effective. Doing this in an open forum also allows easy access and comments by external contributors.

Before creating an RFC, it needs to be clear if an RFC is actually needed. The following describes each situation where an RFC is mandatory:

 - SDK behavior or API changes when interacting with the User OR the Server. So every time either the user or the server communication changes, there is a high likelihood other SDKs have a similar impact. This explicitly includes utility classes
 which are very likely to be needed on other SDKs.
 - The RFC process itself needs to be changed.
 - Existing inconsistencies that need to be resolved across the board.

If a prototype is already ongoing, it makes sense to drive the RFC alongside the prototype, but this is not a requirement.

Two examples: if the java client includes a helper class for retry semantics on RxJava observables (to reduce the verbosity in writing it manually), this does not have an equivalent on any other SDK and therefore doesn't need a RFC. If the java client introduces a helper class for LDAP support, this does have an impact on the other SDKs and needs a RFC.

When it is not clear if an RFC needs to be created, a quick email or discussion in the team should make it clear whether the change has an overall impact or not.

**An RFC MUST be signed off by each SDK representative and consensus must be reached. Once the RFC is accepted, each SDK MUST implement it according to the RFC. If, after acceptance issues show up during implementation, the MUST lead to a revision of the RFC which again needs to be signed off by everyone.**

# General Design

The RFC must always be in one of the following states, and transition through them in the given order:

 1. *Identified*: The need for an RFC has been identified, but none actually drafted yet.
 2. *Draft*: The RFC is currently being drafted and subject to heavy modifications.
 3. *Review*: The RFC is believed to be complete (by stakeholders) and is broadcasted for a wider circle for final remarks. Relevant SDKs need to fully implement the RFC before it can be moved into Accepted.
 4. *Accepted*: The RFC is accepted, no further changes can be made to this RFC version. Subsequent changes need to be lifecycled again.

The following listing outlines the rough steps involved to coordinate a proper RFC cycle. Note that alternatively it has been decided that
during the DRAFT and REVIEW phase of an RFC the interaction can happen on Google Docs, but once it hits ACCEPTED it needs to be moved back
into the RFC repo as outlined below for maximum visibility.

 1. Look at latest drafted RFC to determine the RFC number (4 digits) - the `RFCID` is in the form *`rfcNumber`***-***`rfcShortName`*
 2. open an issue titled `RFCID`, short description of the RFC, does the owner think it requires SDK specifics, etc. (the issue allows to "reserve" the RFC number prior to any file being added publicly to the repo).
 3. The issue is now in IDENTIFIED state - modify the README.md (https://github.com/couchbaselabs/sdk-rfcs/blob/master/README.md) to include it and push it into master. The link of the ticket should reference the github issue, which should reference the PR for the RFC later. This allows for easy discoverability.
 3. create the RFC branch: `drafts/RFCID`
 4. create the RFC file as `RFCID.md`in `/rfc` directory
 5. Write the first version of the RFC.
 6. start the discussion by doing a Pull Request to master
 7. the RFC is now in draft state. Update the README and set the state to Draft.
 8. **phase one** iterates on the global idea + "proof of concept" SDK specifics proposed by the owners.
 9. Stakeholders (SDK Team + different people depending on the RFC) need to sign it before it goes into Review.

 ```
 rfc is (:+1:|:-1:) for `teamname` *anything after is a vote comment*
 ```

 examples:

 ```
 > rfc is :-1: for `java` - I'm not happy with the java specifics after toying with it
 > rfc is :+1: for `python`
 ```

 10. the RFC is now in review state. Assign a minimum review period of one week (managed by the RFC owner), optionally longer.
 11. Each identified SDK that participates in the RFC needs to implement it, at least marked as experimental.
 12. Once there are no objections, the RFC is accepted.
 13. SDK implementation at this point can move from "experimental" to "supported" for artifacts of this RFC.
 12. Merge the RFC into master, making it visible permanently. Update the README to reflect the latest change and link directly to the RFC instead of the issue.

 Once content is :+1: by all teams, merge into master using the following strategy (it retains discussion on intermediate commits, but produces a linear history in master):

```
 - `git merge --squash -e drafts/RFCID`
 - `git commit`
 - commit message with
   - user that signed off for each `team`
   - link to PR: `Close #XX (see link for discussion)` (also closes the pr since github won't detect a merge --squash)
   - `Close #YY` (to close the issue)
```

 13. Issue needs to be closed.

See the template in 0000-template.md.

# Language Specifics
Not applicable in this RFC.

# Signoff
If signed off, each representative agrees both the API and the behavior will be
implemented as specified.

Note that this signoff includes every SDK team member.

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| -        | Michael N.     | 2015-12-03 |
| -        | Simon B.       | 2015-12-03 |
| -        | Brett L.       | 2015-12-09 |
| -        | Jeff M.        | 2015-12-11 |
| -        | Sergey A.      | 2015-12-11 |
| -        | Mark N.        | 2015-12-09 |
| -        | Todd G.        | not voted |
| -        | Matt I.        | not voted |

# Errata

 - *2016-01-20*: Workflow Changes in the Process & States added (Michael N.)
 - *2016-08-22*: Added clarification on action items during REVIEW (Michael N.)
