# Security policy

Thank you for helping keep Signboard and the businesses that use it safe.

## Reporting a vulnerability

Please **do not** open a public issue, discussion or pull request for a security problem.

Report it privately through GitHub: on the repository's **Security** tab, choose **Report a vulnerability**, or go straight to [the private reporting form](https://github.com/signboard-works/signboard/security/advisories/new).

Please include:

- what the problem is, and what an attacker could do with it;
- the steps to reproduce it, or a proof of concept;
- the version, commit or URL affected;
- how you would like to be credited, if at all.

## What to expect

| | Target |
|---|---|
| Acknowledgement | Within 3 days |
| Fix for a critical issue | Within 7 days |
| Fix for a high-severity issue | Within 30 days |
| Other issues | In a normal release |

We will keep you updated, agree a disclosure date with you, and publish a GitHub security advisory (with a CVE where it applies) once the fix is released. Signboard is maintained by one person in their spare time, so these are targets, not guarantees, but security reports always come first.

## Supported versions

Security fixes are made in the latest release only. Self-hosted installations get fixes by upgrading.

## Testing

When testing the hosted service at `signboard.works`, please only use accounts you created yourself, never access or change another business's data, and do not run load or denial-of-service tests. Testing a self-hosted copy of your own is always welcome.

How Signboard approaches security is described in [ADR 0039](https://docs.signboard.works/engineering/decisions/0039-security-baseline/).
