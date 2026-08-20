# Security Policy

## Reporting a vulnerability

If you find a security issue in this project, please report it by opening a
GitHub issue labeled `security`, or by emailing the maintainer through the
address listed on the [GitHub profile](https://github.com/imfaisii).

Please do not include real App Store Connect credentials, `.p8` files, or
other secrets in a public issue.

## Credential model

This server never stores App Store Connect credentials on disk or in the
repository:

- Auth is a short-lived ES256 JWT, built at runtime from `ASC_ISSUER_ID`,
  `ASC_KEY_ID`, and your `.p8` private key (via `ASC_PRIVATE_KEY_PATH` or
  `ASC_PRIVATE_KEY`).
- Tokens are valid for 20 minutes and are cached in memory only — never
  written to disk.
- No API keys, tokens, or `.p8` contents are logged or persisted anywhere
  by this project.

## If you accidentally commit a `.p8` key

Revoke the key in App Store Connect **immediately** (Users and Access ->
Integrations -> App Store Connect API), then generate a new key. Rotating
the key in App Store Connect is the only way to fully invalidate it — removing
it from git history afterward does not undo the exposure.
