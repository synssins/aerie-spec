# Q-0005: Basic auth for accounts with sign-in codes (provisional)

Raised by: the compatibility lab, first run against a real aerie image.
Status: answered in pull request `phase1-rulings-1`.

## Where

Data Center compatible surface: HTTP Basic on every operation. Aerie requires administrators to use time-based sign-in codes, and other users may turn codes on.

## Question

What does the compatible surface do with Basic credentials for an account that has sign-in codes turned on?

## Answer

Provisional. `behaviour/identity.md` is being written in a separate session and will replace this ruling.

- Basic credentials for an account with sign-in codes turned on are refused with `401`. This covers codes the user turned on and codes a policy requires.
- The response is the same as for a wrong password, so Basic cannot confirm a password while skipping the code. Aerie logs the real reason to its audit log.
- Such accounts use a personal access token. Accounts without codes, or exempted by an administrator, use Basic as before.

Written in `behaviour/common.md` (Authentication, Accounts with sign-in codes), with the `basic` security scheme description in `contracts/datacenter/openapi.yaml`.
