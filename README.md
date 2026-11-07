# cdkd crashes on startup with `@aws-sdk/client-kinesis` 3.1018.0–3.1138.x

## Summary

cdkd needs `@aws-sdk/client-kinesis` **3.1139.0 or later**, but its `package.json` only asks for **3.1018.0 or later** (`^3.1018.0`).

If a project ends up with a version between 3.1018.0 and 3.1138.x, every cdkd command crashes on startup.

| `@aws-sdk/client-kinesis` version | Allowed by cdkd's `package.json`? | Does cdkd work? |
| --- | --- | --- |
| 3.1017.x or older | No | — |
| **3.1018.0 – 3.1138.x** | **Yes** | **No, it crashes** |
| 3.1139.0 or later | Yes | Yes |

## Why it happens

- cdkd imports `UpdateStreamRecordDistributionStrategyCommand` from `@aws-sdk/client-kinesis` in [`src/provisioning/providers/kinesis/update.ts`](https://github.com/go-to-k/cdkd/blob/main/src/provisioning/providers/kinesis/update.ts).
- That command first appeared in `@aws-sdk/client-kinesis` 3.1139.0 ([CHANGELOG](https://github.com/aws/aws-sdk-js-v3/blob/main/clients/client-kinesis/CHANGELOG.md)).
- With an older version, the import fails as soon as cdkd loads.

## When it happens in real projects

When cdkd is installed on its own, the package manager picks the newest `@aws-sdk/client-kinesis` (3.1139.0 or later), so you won't see the problem.

It only shows up when another dependency has already installed an older version. For example, `@aws-amplify/analytics` can pull in 3.1106.0. Because 3.1106.0 is within cdkd's declared range, the package manager reuses it for cdkd. It is allowed to do this, and cdkd crashes as a result.

This repository adds `@aws-sdk/client-kinesis@3.1106.0` as a direct dependency to create that situation.

## How to reproduce

You don't need an AWS account. The crash happens before cdkd calls AWS.

```sh
pnpm install --ignore-workspace
AWS_REGION=us-east-1 AWS_ACCESS_KEY_ID=dummy AWS_SECRET_ACCESS_KEY=dummy \
  pnpm exec cdkd diff --all
```

### What happens

```
SyntaxError: Named export 'UpdateStreamRecordDistributionStrategyCommand' not found. The requested module '@aws-sdk/client-kinesis' is a CommonJS module, which may not support all module.exports as named exports.
```

### What should happen

cdkd starts and reaches the AWS call. The dummy credentials are rejected, which is expected:

```
InvalidClientTokenId: The security token included in the request is invalid.
```

To see this, change `@aws-sdk/client-kinesis` in `package.json` to `3.1139.0` and run the steps again.

## Versions tested

- `@go-to-k/cdkd` 0.312.0
  - This package is deprecated and has moved to `@cdkd/cli`.
  - It is used here because every `@cdkd/cli` release is less than 7 days old (first published on 2026-11-03), and the test environment blocks packages that new with pnpm's `minimumReleaseAge` setting.
  - The latest `@cdkd/cli` (0.333.0) still declares `^3.1018.0`, and its source still imports the command, so it is expected to have the same problem. This has not been run yet.
- `@aws-sdk/client-kinesis` 3.1106.0 (crashes) and 3.1139.0 (works)
- Node.js 24.22.1, pnpm 12.16.1
