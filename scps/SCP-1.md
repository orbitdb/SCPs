
## SCP-001: SCP Categories and Nomenclature

### SCPs

MUST pertain to several different categories:

1. *Architecture* SCPs MUST relate to the internal architecture and software engineering of OrbitDB. Proposals regarding performance, resiliency, and feature sets are submitted in this category.
2. *DevEx* SCPs MUST relate to the general DevExp a contributor has when _developing_ OrbitDB itself. Proposals regarding CI/CD, tests, linting, and *internal* documentation are filed here.
3. *UI/UX* SCPs MUST relate to the general Look and Feel/Experience a User has when _using_ OrbitDB tools. Proposals regarding UI, UX and tests are submitted in this category.
4. *Security* SCPs MUST relate to security in the OrbitDB environment. Proposals related to improvements in distribution, data transmission, encryption and access control are filed in this category.
5. *Community* SCPs MUST relate to the external documentation, code of conduct, community engagement, and the SCP process itself.


### Anatomy

1. SCPs MUST have a header made up of the format SCP-{PR}: followed by the title of the SCP. 
   eg: **SCP-002: SCP Proposal**. [See TEMPLATE.md](../TEMPLATE.md)

### Concensus

The decision by consensus MUST be evaluated according to vote in the response given to the PRs.

1. People SHOULD respond with their arguments for *why yes* or *why no*.
2. Votes to response MUST be given as *thumbs up* for +1 or *thumbs down* for -1 and the total MUST be the result of subtraction of *thumbs up* - *thumbs down*.
3. The winning response is based on the votes, the proposal MUST be accepted by consensus if the response argument was for "yes" or rejected if it was for "no".

Note: Proposals MAY be vetoed by the maintainers if it is determined by consensus among themselves that the proposal violates is out of scope of the above categories, or violates any type of law against e.g. privacy or copyright, or if the proposal violates other community standards.


### Numbering

SCPs MUST be numbered the same as the pull request which they are introduced.
eg. SCP-002: SCP Proposal

Following RFC-2119:
https://datatracker.ietf.org/doc/html/rfc2119
