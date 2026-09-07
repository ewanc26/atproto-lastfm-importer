#AGENTS.md

Guidance for AI coding agents working in this repository. Human contributors
may find it useful too, but the audience is agents.

## Project overview

A native C/C++23 Minecraft: Java Edition server focused on predictable, low RAM
usage. Zincfox is an experimental clean-room server implementation: the goal is
not to clone the vanilla server architecture in C++, but to build the protocol,
simulation, world and persistence layers around explicit ownership, bounded
queues and measurable memory budgets from the start.

- **Language:** C17 is available for small leaf components where it reduces
  runtime/dependency surface; C++23 is the default for protocol, server,
  storage, and world state. `snake_case` for functions and variables,
  `PascalCase` for types.
- **Build:** CMake, C17/C++23 strict by target, `-Wall -Wextra -Wpedantic
  -Wconversion -Wsign-conversion`. Tests are per-file executables run through `ctest`,
  following the account's other native repos (`clay/`, `wolfram/`, `keepsake/`).
- **Target:** macOS and Linux desktop. Windows is untested (as elsewhere in this
  account).

## Repository layout

```
include/zincfox/       public/internal C/C++ interfaces
src/protocol/          VarInt, framing, packet/state codecs
src/server/            connection lifecycle and dispatch
src/world/             world/chunk state (future)
src/entity/            entity/player storage (future)
src/storage/           region/persistence backends (future)
test/                  unit and protocol regression tests
docs/                  design notes and compatibility records
```

Dependency direction is inward from higher-level game/server code to small
protocol/net abstractions. Do not let world/entity code call raw socket APIs.

## Module boundaries — read before editing

- **Protocol code owns all wire-format parsing.** `src/protocol/` must stay
  free of server lifecycle concerns;
`src / server /` must stay free of game -
        state concerns
            .The boundary is the `protocol::handle_packet` dispatch interface.-
        **Version -
        specific packet definitions stay in `src /
            protocol /`.**Transport and game systems must not accumulate packet
                              IDs or
    version checks.Put version tables /
            codecs behind the protocol layer so supporting another Minecraft
                release does not fork the whole server.-
        **Connection state is owned by `src /
            server /`.**The protocol layer sees only
                            borrowed `std::span` payloads; it must not retain decoded packet objects
  after dispatch.
- **No global mutable server state.** A subsystem that owns a thread must
  expose shutdown/join semantics and memory/queue bounds.

## Build and run

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
ctest --test-dir build --output-on-failure
./build/zincfox [--port 1-65535]
```

"Verified" means: clean build (zero warnings under the strict flags), `ctest`
green, and — for anything touching the network path — a real client connection
path for the claimed states with automated regression fixtures retained where
licensing permits.

## Configuration

- **All configurable behavior belongs in the global `zincfox.conf` file.** Do
  not add hidden environment flags, command-only switches, or per-module
  configuration files for server behavior. A new setting must have a bounded
  type/range, a documented default, load/save coverage, and an explanation of
  its retained-memory or resource effect when relevant.
- Configuration must never make an unbounded queue, cache, world, or player
  store possible. Dynamic choices must resolve to one of documented finite
  limits and select the safe lower limit when host information is unavailable.

## Versioning

- Releases use strict semantic versioning `v<major>.<minor>.<patch>`.
- The version lives only in the `VERSION` line of `CMakeLists.txt`; derive any
  runtime version string from that single source of truth, not a separate file.
- **No version jumps**: bump from the immediately previous released version.
  Never skip a patch, minor, or major number; do not backfill gaps with phantom
  tags or releases.
- **Substantial changes require a release cut**: a user-visible protocol or
  gameplay behavior, persistence/world-format change, compatibility claim,
  public interface change, or material resource-budget change must not be
  allowed to accumulate indefinitely after a release. Before merging the next
  substantial tranche, audit the commits since the latest tag and cut the next
  sequential version when the tranche is ready. Documentation-only, test-only,
  formatting, and internal refactors do not require a version cut unless they
  change the published contract.
- **Release procedure follows Wolfram**: change the single `VERSION` line,
  create a signed annotated `v<major>.<minor>.<patch>` tag on that same commit
  (falling back to an annotated tag only when signing is unavailable), push the
  commit and tag, and create the matching GitHub release with generated notes.
  For pre-1.0 releases, publish source only; attach built artifacts starting at
  `v1.0.0`.

## Code style

- Header guards (`ZINCFOX_PROTOCOL_<FILE>_HPP`), not `#pragma once` — matches
  the convention in `wolfram/include/wolfram/` and `clay/include/clay/`.
- `.clang-format` in this repo (LLVM base, 4-space indent, 80 columns,
  attached braces) — run `clang-format -i` on changed files.
- Comments explain *why*, sparingly; never narrate obvious code.
- No C++ exceptions for expected protocol/server states. Use explicit
  result/error types. Reserve exceptions/aborts for genuine programmer errors.
- Avoid RTTI-heavy or virtual object hierarchies for packets/entities when
  tagged values or tables are simpler.

## Memory invariants

The initial scaffold deliberately chooses simple fixed bounds:

- 32 connection slots;
- one 8 KiB receive buffer per slot;
- one 128 KiB transmit buffer per slot (sized for one columnar 24-section
  chunk frame with full sky light);
- one small protocol / session record per slot;
- one `pollfd` table for the listener plus those slots.

The fixed socket-buffer payload is therefore **4.25 MiB** at maximum connection
capacity (32 slots x 136 KiB), plus small connection/poller metadata and
operating-system socket buffers. This is not a promise that the process RSS is
4.25 MiB, but it is the first explicit retained-memory budget owned by Zincfox
itself.

When adding a subsystem, document its steady-state and worst-case retained
memory in the PR when practical.

Every long-lived subsystem should answer four questions:

1. What owns this memory?
2. What is the normal retained size?
3. What is the maximum retained size or eviction/backpressure rule?
4. What input can cause the subsystem to grow?

## Commits and pull requests

Matches the convention in `wolfram/AGENTS.md` / `keepsake/AGENTS.md`.

- **Atomic conventional commits**: every commit is exactly one logical change.
  Scope by module — `feat(protocol)`, `feat(server)`, `fix(net)`,
  `test(protocol)`, etc. Never combine a code change with a docs update, or
  changes to two unrelated modules, in one commit. Write the message to explain
  the reasoning, not just restate the file list. Split multi-concern work into
  sequential commits instead.
- **Metadata files may be updated directly on `main`.** This covers project-level
  metadata and documentation such as `AGENTS.md`, `README.md`, `docs/**`, and
  similar non-code files that guide how the repository is maintained.
- **All other work lands via feature branches and pull requests.** Code,
  tests, build scripts, and any behavioral change must be developed on a
  dedicated `feat/<area>` or `fix/<area>` branch and merged through a PR so
  review and CI run before it reaches `main`.
- **Honest attribution**: commits may carry a `Co-authored-by:` trailer crediting
  an AI agent, and may reference the specific model used, in the commit message,
  a PR, or code comments — attribution should reflect who/what actually did the
  work.
- **No commented-out code** left in place; delete dead code or move it to a
  test.

## Issue tracking

- **Track every discovered issue**: a bug, protocol mismatch, portability
  defect, missing test, documentation inconsistency, or deferred compatibility
  problem found during development or review must have a GitHub issue unless it
  is fixed in the same atomic change and leaves no follow-up work.
- Create issues with the repository templates under
  `.github/ISSUE_TEMPLATE/` (`bug_report.yml` for defects and
  `feature_request.yml` for requested behavior). Include the exact version or
  commit, reproduction or evidence, affected protocol state, and relevant
  test/CI output. Do not substitute private notes or an untracked TODO for a
  reportable issue.
- Link the issue from the implementing pull request and close it only when the
  fix or explicitly scoped follow-up has been verified. Release audits must
  review open issues before declaring a tranche complete.

## Do not do these without explicit human sign-off

- Add a JVM/Paper/Spigot server as the actual backend.
- Copy Mojang proprietary server source or decompiled implementation code.
- Add an unbounded network/task/chunk queue.
- Replace protocol validation with permissive "best effort" parsing.
- Introduce a dependency-heavy game/server framework.
- Claim vanilla compatibility for a release without client/protocol tests.
- Weaken warnings, sanitizers or tests merely to get CI green.
