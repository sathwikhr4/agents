# Server test tree typecheck

Status: the ratchet is in place. The backlog removal has not started.

## Problem

`server/tsconfig.json` excludes `server/src/__tests__`, which has 1,101 TypeScript
files. No script or CI job referred to `server/src/__tests__/tsconfig.json`. That
config exists only because the test transform needs a nearby tsconfig (#13141).
Vitest removes types and does not check them.

Only 27 of the 1,101 files are in the program of another tsconfig, and these 27 have
0 errors. Most of them are helpers that other tests import. 11 of them are in
`helpers/maintenance/tsconfig.json`, and no script runs that project. So no tool
checked the other 1,074 files.

## Error breakdown

The measurement uses TypeScript 7.0.2. Each row adds one change to the row above.

| Step | Total errors | Change | Cause |
| --- | --- | --- | --- |
| Config as committed | 6,536 | | |
| Add `src/types/*.d.ts` | 4,447 | -2,089 | Config. The Express request types were missing. |
| Set `rootDir` to the repository root | 2,436 | -2,011 | Config. Tests import files outside the old `rootDir`. |
| Set `allowImportingTsExtensions` | 2,181 | -255 | Config. Vitest accepts `.ts` imports. |
| Add `ES2024.Promise` to `lib` | 2,138 | -43 | Config. Node 24 has `Promise.withResolvers`. |
| Set `allowJs` for `.mjs` imports | 2,148 | +10 | Config. It removes 8 `TS7016` errors and shows new real errors. |
| Set `declaration` to false | 2,142 | -6 | Config. Tests emit nothing. |
| Count only files in `server/src` | 2,058 | -84 | Other packages. See below. |

Result: 4,478 of the 6,536 errors (69%) came from config or scope. 2,058 are real
type errors, in 314 of the 1,101 files.

The 84 errors in other packages:

- 53 are in the Daytona plugin. The root install does not install its dependency.
  With that dependency installed, these errors go away and the test tree count does
  not change.
- 29 are in `packages/paperclip-runner/src/contracts/native-execution.test.ts`.
  The Runner tsconfig excludes `*.test.ts`, so this is a similar gap. This plan does
  not cover it.
- 2 are in `ui/src/lib/issue-execution-policy.ts`.

The 2,058 real errors by kind:

| Kind | Errors | Codes |
| --- | --- | --- |
| Wrong argument or assignment type | 1,023 | `TS2345`, `TS2322`, `TS2769`, `TS2554` |
| Stale or partial fixture objects | 525 | `TS2741`, `TS2739`, `TS2740`, `TS2353`, `TS2352` |
| Property that the type does not have | 317 | `TS2339`, `TS2551`, `TS2493` |
| Missing narrowing of null or unknown | 101 | `TS18046`, `TS18047`, `TS18048`, `TS2532`, `TS2722` |
| Implicit any | 60 | `TS7006`, `TS7005`, `TS7034`, `TS7031` |
| Missing name or module | 16 | `TS2304`, `TS2307` |
| Other | 16 | |

A real type error does not always mean a wrong test. Most of these errors are stale
fixtures and loose mocks. Some of them hide real problems:

- `chat-channels.integration.test.ts` calls `service.test(endpoint.id, "owner-user")`
  at about 100 sites. The second parameter is
  `{ optionalSlackTestForUser: string }`, and a string has no such property. So the
  service ignores the argument, and the tests pass without the option that they
  appear to send.
- `generic-mcp-connection.test.ts` imports `../services/composio.js`. #13758 removed
  that module. The test passes because the transform drops the unused imports.

## Why a ratchet and not one pull request

314 files is too large a review for one pull request. It would also conflict with
most other server work, because these test files change often. The ratchet stops
new errors now, and the backlog can go down in small pull requests.

## What is in place

- `server/src/__tests__/tsconfig.json` has the correct options, so editors and the
  typecheck agree.
- `server/scripts/typecheck-layers.mjs` checks the tree in 3 shards and reads the
  server as declarations. Each shard peaks at less than 2.5 GiB on one thread. One
  program over the tree peaks at 3.9 GiB. All of these results have the same errors.
- `server/scripts/typecheck-test-tree-baseline.json` holds the error count per file.
  A count above the baseline fails. A count below the baseline fails until the
  author lowers the baseline with this command:

  ```sh
  pnpm --filter @paperclipai/server typecheck --update-test-baseline
  ```

  That command never raises a count or adds a file. A raise must be an edit by hand,
  so it shows in the diff.

## Limits

- The baseline counts errors per file. An author can fix one error and add a
  different error in the same file, and the check passes.
- A file with 0 errors gets the full protection at once. A file in the baseline gets
  it only when its count goes to 0.

## Remove the backlog

Do these in order. Each step is one pull request. Each step lowers the baseline.

1. **Fix the cases that can hide test problems.** Decide what each
   `service.test(endpoint.id, "owner-user")` call must send (108 errors). Remove the
   imports of deleted modules (`TS2304`, `TS2307`).
2. **Fix the shared fixtures.** One helper or type change can remove many errors:
   - `Environment.envVars` (73)
   - `DuplexChannelOpenInput` fixtures (41)
   - `string | null` to `string | undefined` arguments (63)
   - `mock.calls` tuple indexes (48)
   - Express `Request` mocks (30)
   - schedule trigger fixtures (27)
   - `IssueWithLabelsAndRun` attention fields (33)
   - `runtimeGeneration` on chat setup state (23)
3. **Clear the small files.** 172 files have 1 or 2 errors. A file with 0 errors
   leaves the baseline and gets the full protection.
4. **Clear the large files.** 10 files hold 813 errors.
   `chat-channels.integration.test.ts` has 300, which step 1 reduces.
5. **Remove the ratchet.** When the baseline is empty, delete it and the comparison
   code. The test-tree layers then fail on any error, as the other layers do.
