# cdkd: `@aws-sdk/client-kinesis` lower bound is too low

`@go-to-k/cdkd@0.312.0` fails at startup when `@aws-sdk/client-kinesis` resolves to a version below 3.1139.0, even though that version satisfies the declared range `^3.1018.0`.

## Cause

- cdkd imports `UpdateStreamRecordDistributionStrategyCommand` from `@aws-sdk/client-kinesis` (`src/provisioning/providers/kinesis/update.ts`).
- That command was added in `@aws-sdk/client-kinesis` **3.1139.0** ([CHANGELOG](https://github.com/aws/aws-sdk-js-v3/blob/main/clients/client-kinesis/CHANGELOG.md)).
- cdkd declares `"@aws-sdk/client-kinesis": "^3.1018.0"`, so a package manager may legitimately reuse an older version that already exists in the dependency tree. In real projects this happens when another dependency (for example `@aws-amplify/analytics`, which declares `^3.1012.0`) has already resolved an older client.

This project pins `@aws-sdk/client-kinesis@3.1106.0` as a direct dependency to simulate that situation.

## Reproduce

No AWS account is needed. The error happens while modules load, before any AWS call.

```sh
pnpm install --ignore-workspace
AWS_REGION=us-east-1 AWS_ACCESS_KEY_ID=dummy AWS_SECRET_ACCESS_KEY=dummy \
  ./node_modules/.bin/cdkd diff --all
```

`pnpm why @aws-sdk/client-kinesis` shows that cdkd uses the hoisted 3.1106.0:

```
@aws-sdk/client-kinesis@3.1106.0
├─┬ @go-to-k/cdkd@0.312.0
│ └── cdkd-kinesis-repro (dependencies)
├─┬ cdk-local@0.158.0
│ └── @go-to-k/cdkd@0.312.0 [deduped]
└── cdkd-kinesis-repro (dependencies)
```

### Actual

```
SyntaxError: Named export 'UpdateStreamRecordDistributionStrategyCommand' not found. The requested module '@aws-sdk/client-kinesis' is a CommonJS module, which may not support all module.exports as named exports.
```

### Expected

cdkd loads and reaches the AWS call. With dummy credentials that means:

```
InvalidClientTokenId: The security token included in the request is invalid.
```

You get this result after changing `@aws-sdk/client-kinesis` in `package.json` to `3.1139.0` and reinstalling.

## Versions

- `@go-to-k/cdkd` 0.312.0. `@go-to-k/cdkd` is deprecated ("moved to @cdkd/cli"). `@cdkd/cli@0.333.0` still declares `^3.1018.0`, and `main` still imports the command.
- `@aws-sdk/client-kinesis` 3.1106.0 (fails) and 3.1139.0 (works)
- Node.js 24.22.1, pnpm 12.16.1

## Suggested fix

Raise the declared range to `^3.1139.0` and check the lower bounds of the other `@aws-sdk/client-*` dependencies against the APIs they import.
