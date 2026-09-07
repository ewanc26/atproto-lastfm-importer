# Contributing to Zincfox

Zincfox is an experimental clean-room C/C++23 Minecraft: Java Edition server.
The current implementation has an experimental protocol-767 login,
configuration, and Play-spawn path validated with the MCP client, but no
general real-client, version, or gameplay compatibility claim.

## Before submitting changes

Build and test with the repository's strict warnings enabled:

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
ctest --test-dir build --output-on-failure
cmake -S . -B build-san -DCMAKE_BUILD_TYPE=Debug \
  -DZINCFOX_ENABLE_SANITIZERS=ON
cmake --build build-san -j
ctest --test-dir build-san --output-on-failure
```

Run `clang-format` on changed C/C++ files. New protocol behavior needs positive,
malformed-input, fragmentation, and boundary coverage. Network behavior must
remain non-blocking and bounded; prefer fixed-capacity storage, reusable
buffers, borrowed `std::span` inputs, and explicit ownership.

For the real-client regression path, set `ZINCFOX_MCP_ROOT` to the local
`mcp-minecraft` checkout and run the server on `127.0.0.1:25565`:

```sh
ZINCFOX_MCP_ROOT=/path/to/mcp-minecraft node test/client_regression.mjs
```

The harness uses two 1.21.1 clients, verifies both reach Play spawn, checks
unsigned system-chat delivery, and exercises movement, terrain dig/place, and
disconnect broadcasts. It uses unique bounded usernames and derives interaction
coordinates from the generated surface so persisted player state cannot make a
run accidentally pass or fail. It is a local development check because the MCP
dependency is intentionally not vendored.

## Protocol and compatibility

Protocol definitions belong in `src/protocol/` and version-specific behavior
must remain isolated from transport and game state. Public references used for
wire formats must be recorded in the change or its documentation. Do not copy
Mojang code or claim a Minecraft version until a real client path and automated
regression coverage exist.

Every new long-lived allocation or queue must document its owner, normal size,
maximum size, and growth/backpressure rule. The initial networking budget is
32 connection slots with fixed 8 KiB receive and 128 KiB transmit buffers per slot.
User-visible errors and connection drops must use a unique hexadecimal code;
see `docs/error-codes.md`.

All configurable server behavior belongs in the global `zincfox.conf` file.
New settings need a validated finite range, a documented default, load/save
tests, and memory/resource documentation where applicable. Dynamic settings
must resolve to documented finite limits when host information is unavailable.

## Commits and pull requests

Use a dedicated `feat/<area>` or `fix/<area>` branch; never push feature work
directly to `main`. Make atomic conventional commits such as
`feat(protocol): ...`, `fix(net): ...`, or `test(protocol): ...`. Pull
requests should explain compatibility claims, memory bounds, test commands,
portability, and any borrowed design or reference material.

## Releases

Zincfox uses strict semantic versions and cuts the next sequential release when
a substantial tranche is ready. Substantial means a user-visible protocol or
gameplay change, persistence/world-format change, compatibility claim, public
interface change, or material resource-budget change; documentation-only,
test-only, formatting, and internal refactoring changes do not require a
release unless they alter the published contract.

Before cutting a release, audit the commits since the latest tag. Update only
the `VERSION` line in `CMakeLists.txt`, commit that bump with the finished
tranche, create a signed annotated `v<major>.<minor>.<patch>` tag on the same
commit (or an annotated tag if signing is unavailable), push both, and create a
GitHub release with generated notes. Releases before `v1.0.0` are source-only;
release artifacts begin with `v1.0.0`. Never skip a version or create a tag or
release without its matching version commit.

Zincfox is licensed under the GNU Affero General Public License v3.0. Keep
license notices and attribution intact when using external references or
borrowed designs.
