# Meta

 - RFC Name: The RFC Process
 - RFC ID: 0001
 - Start Date: 2015-10-23
 - Owner: Michael Nitschinger
 - Current Status: Review

# Summary
This RFC describes the newly established RFC process.

# Motivation
Since the inception of the "SDK", we struggled with inconsistencies between the different language implementations. Many different reasons are contributing to it (legacy codebases, customer escalations, forgetting to communicate,...), but in the end there are two noticable problems to the user:

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

Two examples: if the java client includes a helper class for retry semantics on rxjava observables (to reduce the verbosity in writing it manually), this does not have an equivalent on any other SDK and therefore doesn't need a RFC. If the java client introduces a helper class for LDAP support, this does have an impact on the other SDKs and needs a RFC.

When it is not clear if an RFC needs to be created, a quick email or discussion in the team should make it clear whether the change has an overall impact or not.

**An RFC MUST be signed off by each SDK representative and consensus must be reached. Once the RFC is accepted, each SDK MUST implement it according to the RFC. If, after acceptance issues show up during implementation, the MUST lead to a revision of the RFC which again needs to be signed off by everyone.**

# General Design

 1. Look at latest issue to determine the RFC number (4 digits) - the `RFCID` is in the form *`rfcNumber`***-***`rfcShortName`*
 2. open an issue titled `RFCID`, short description of the RFC, does the owner think it requires sdk specifics, etc... tag as appropriate (if we decide to put tags in). the issue allows to "reserve" the RFC number prior to any file being added publicly to the repo.
 3. create the RFC branch: `drafts/RFCID`
 4. create the RFC file as `RFCID.md`in `/rfc` directory
 5. start the discussion by doing a Pull Request to master
 6. **phase one** iterates on the global idea + "proof of concept" sdk specifics proposed by the owner
 7. *end of this phase should probably be marked by a first informal round of votes?*
 8. **phase two** iterates on SDK specifics: each SDK team/representative can then push commits in the branch to improve on his own SDK specifics
 9. once everyone has contributed SDK specifics, the RFC must be signed off by a message following this convention:

 ```
 rfc is (:+1:|:-1:) for `teamname` *anything after is a vote comment*
 ```

 examples:

 ```
 > rfc is :-1: for `java` - I'm not happy with the java specifics after toying with it
 > rfc is :+1: for `python`
 ```

Once content is :+1: by all teams, merge into master using the following strategy (it retains discussion on intermediate commits, but produces a linear history in master):

 - `git merge --squash -e drafts/RFCID`
 - `git commit`
 - commit message with
   - user that signed off for each `team`
   - link to PR: `Close #XX (see link for discussion)` (also closes the pr since github won't detect a merge --squash)
   - `Close #YY` (to close the issue)

See the template in 0000-template.md.

# Language Specifics
Not applicable in this RFC.

# Signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

Note that this signoff includes every SDK team member.

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| -     | Michael N. | 2015-12-03 |
| -     | Simon B. | 2015-12-03 |
| -     | Brett L. | 2015-12-09 |
| -     | Jeff M. | 2015-12-11 |
| -     | Sergey A. | 2015-12-11 |
| -     | Mark N. | 2015-12-09 |
| -     | Todd G. | not voted |
| -     | Matt I. | not voted |
