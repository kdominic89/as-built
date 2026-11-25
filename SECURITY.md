# Security

## Supported versions

Only the latest published version of the image is corrected.
A correction is published as a new version; a published version is never
replaced under the same name.

## Reporting a vulnerability

Report it privately, not as a public issue:
open the **Security** tab of this repository and choose
**Report a vulnerability**.

Please include:

- the version or the digest of the image,
- the command you ran,
- what happened, and what you expected instead.

A report that needs a tree to reproduce it is most useful with the smallest
tree that still shows the problem.
Do not attach code you may not share.

## What the program touches

The program reads the repository it is pointed at and the package folder you mount,
and it writes the one architecture document when asked.
It opens no network connection, so it can always be run with `--network none`.
