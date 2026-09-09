# ancaria workspace root

## Repository purpose

This is the organisation’s `.github` repository. It holds the workspace
files that join the project repositories and nothing else. It contains no
application code. The [ancaria.dev](https://ancaria.dev) front page is its own
repository, `site`.

## Project architecture

Sacred was released in 2004. The Sacred Gold compilation followed in 2005.
ancaria is a mod loader for Sacred Gold. Mods are written in Java against an
event API and loaded while the game is running.

Every address in the project targets `pureHD.exe` 2.0.2.118, a 32-bit community
HD wrapper with image base `0x00400000`. Those addresses do not target the stock
`Sacred.exe`. The loader’s JVM is 64-bit and cannot run inside the 32-bit game
process, so the Rust host runs it as a separate process and carries events
between the game and Java.

The launcher, host, and agent locate the game by trying `pureHD.exe`,
`Sacred.exe`, then `Game.exe`, without regard to case. The address table still
belongs to the supported `pureHD.exe` build. The launcher reports the detected
build before Play. On attach, the agent compares the bytes at its 20 hook sites
with the checked-in signatures and prints a console warning when they differ.
Attaching to another executable name or build is allowed, but the warning does
not make its addresses safe.

    launcher   Installs the loader into the game folder and starts the host
    protocol   Rust host that injects the agent with Frida, starts the JVM,
               and routes frames in both directions
    agent      JavaScript inside the game that hooks the game’s instructions
    zygote     JVM-side loader that loads mod jars and dispatches events
    mod        Java compiled against the coderpack API

Observed events travel from agent to host to JVM asynchronously. A cancelable
event travels as an `ASK` and stops the game thread while the mod decides. The
host’s verdict deadline is 250 ms, but its watchdog scans every 125 ms and
marks an ask overdue only after its age exceeds the deadline. The fallback
`ok` verdict is therefore queued roughly 250 to 375 ms after the ask, plus
scheduler and poster-loop delay. Keep cancelable handlers short and never
describe 250 ms as a strict worst-case block time.

After installation, `<Sacred Gold>/launcher/` contains the host, jars, agent,
and `VERSION` file. Mods live in `<Sacred Gold>/mods`.

The launcher looks for a JDK 21 or newer in `<Sacred Gold>/launcher/java`,
then `JAVA_HOME`, then `PATH`. It continues past an older JDK and uses the
first suitable one. If none is suitable, the launcher and game still run, but
mods do not load. Get Java downloads a selected JDK into
`<Sacred Gold>/launcher/java`. The launcher passes the chosen executable to
the host with `--java`. It does not change `PATH`, write Java registry keys, or
install the JDK outside the game folder.

The loader does not patch the game files on disk. Hooks exist only in the
running process and disappear when it exits. The project is for single-player
use and requires a legally owned, installed copy of Sacred Gold. It provides no
multiplayer features, DRM bypass, game executable, or redistributed game files.

## Repository ownership

The project has nine component repositories at
`https://github.com/ancaria-dev/<name>.git`, all included here as submodules,
including `idea`, the source for the IntelliJ IDEA plugin. Its remote exists
but is still empty; a clone taken before `idea/` is pushed gets an empty
checkout for it.

| Directory | Owns |
|---|---|
| `mappings` | The address registry for `pureHD.exe` 2.0.2.118, including each VA, RVA, and confidence level. |
| `research` | Disassembly scripts, live probes, and research notes. Nothing here ships to players. |
| `coderpack` | The Frida agent in `agent/src`, the Java API in `api`, the JVM-side loader in `zygote`, and the Python tools that read the address registry. |
| `protocol` | The wire protocol and Rust host. It builds `protocol.exe`. |
| `launcher` | The Go executable placed in the game folder. It embeds everything it installs. |
| `build` | The Gradle plugin, mod linter, and `coderpack` project scaffolder. `build/maven` currently contains design notes only. |
| `mods` | The default SRML repository, its index, and the source for four mods. |
| `idea` | The IntelliJ IDEA plugin, including the New Project wizard, Run Sacred configuration, gutter icons, and loader settings. Its workflow is prepared for GitHub and JetBrains Marketplace publication. |
| `site` | The ancaria.dev front end: React, Vite, and LESS modules, deployed to Cloudflare by its own CI. It reads no sibling checkout. |

`ancaria.code-workspace` opens the workspace root and all nine project
directories in one VS Code window. It hides those directories under the root so
they do not appear twice in the file tree.

Edit the repository that owns the change. Put addresses in `mappings`, hooks in
`coderpack`, and investigative probes in `research`. When a probe establishes
an address or fact used by the loader, record the result in `mappings`.

## Checkout

```text
git clone --recurse-submodules https://github.com/ancaria-dev/.github.git
git submodule update --init --recursive     # if cloned without the flag
```

The root repository and all nine submodules use `master`. Each entry in
`.gitmodules` pins `branch = master`.

`idea` is pushed last in the publish order. A recursive clone taken before
that push gets an empty checkout for it; run
`git submodule update --remote idea` again once it has commits.

A full workspace is optional. Side-by-side checkouts provide direct source
coupling. `coderpack` can generate its address table from `../mappings`,
`protocol` can bundle the agent from `../coderpack`, and `launcher` can build
and stage both sibling projects without waiting for published artifacts.

## Independent builds

Each repository can be developed without cloning the complete workspace.

- `coderpack` locates the registry in this order: a command-line path,
  `$CODERPACK_MAPPINGS`, the sibling `../mappings`, then
  `https://raw.githubusercontent.com/ancaria-dev/mappings/<ref>/mappings.json`.
  Downloads are cached under `build/mappings/`. The ref comes from
  `coderpack/.mappings-ref`, currently `master`. Use a tag or commit there when
  the build must be reproducible. See `coderpack/tools/paths.py`.
- `protocol` tests its bundler against `protocol/tests/agent/`, a fixture owned
  by that repository. Its end-to-end test searches for the newest coderpack
  `api` and `zygote` jars in `../coderpack/*/build/libs` and then in
  `~/.m2/repository/dev/ancaria/coderpack/`. It reports a skip when neither
  location contains both jars, so the Rust build itself does not require a JDK.
- `launcher` builds its `protocol` and `coderpack` siblings from source when
  they are available. Otherwise it downloads the releases pinned in
  `launcher/dependencies.json`, currently `protocol` and `coderpack` at
  `0.99.0`. The downloaded files are `protocol.exe`, `api.jar`, `zygote.jar`,
  and `agent.zip`. The zip already contains the generated address table. Run
  `pwsh tools/build.ps1 -Protocol none -Coderpack none` to force this path.
- `idea` resolves the scaffolder as `dev.ancaria.coderpack:templates` from
  Maven Central, like any other dependency. Run `publishToMavenLocal` in
  `build` to test an unreleased template change; `mavenLocal()` is checked
  first in `idea/build.gradle.kts` and overrides the released artifact when
  present.
- `build`, `mappings`, and `research` read no sibling checkout. `mods`
  resolves the plugin and the API through the Gradle Plugin Portal and Maven
  Central, and downloads the `coderpack` command line from the `ancaria-dev/build`
  release pinned in `mods/dependencies.json` for `coderpack index --check`.
- `launcher` ships no mods in its payload. At run time, players choose mods from
  a visible SRML repository. The default is `mods`.

A repository that needs a GitHub Release asset from another repository -- not
a Maven coordinate, which a build tool already versions -- pins it in a
`dependencies.json` at its root: `[{ "path": "ancaria-dev/<repo>", "version":
"<version, no v prefix>" }]`. `launcher` and `mods` both have one. Never
download "latest": a CI step reads the pinned version and asks for that exact
release tag, so a bad release elsewhere cannot break this repository's build
on its own schedule, and a sibling checkout still always wins over the pin
when one is present. This is the same file shape everywhere on purpose, so one
Renovate custom manager (see this repository's `renovate.json`) can bump every
repository's pins.

## Cross-repository build order

Rebuild a cross-repository change in dependency order:

0. When the scaffolder or a template changed, run
   `cd gradle && ./gradlew publishToMavenLocal` in `build` so `idea` and `mods`
   pick it up locally ahead of a release.
1. In `mappings`, run `python mappings.generator.py`, then
   `python mappings.generator.py --check`. This produces `mappings.json`.
2. In `coderpack`, run `python tools/addr.py` to regenerate
   `agent/src/gen/addr.js`. Run `python tools/hooksafe.py` as well for every new
   hooked row.
3. In `coderpack`, run `./gradlew build`. The jars are written to
   `api/build/libs/api-*.jar` and `zygote/build/libs/zygote-*.jar`. CI packages
   the generated agent separately as the release asset `agent.zip`.
4. In `protocol`, run `cargo build --release` to produce
   `target/release/protocol.exe`.
5. In `launcher`, run `pwsh tools/build.ps1`. It rebuilds and stages the sibling
   outputs, reruns address generation, and produces
   `dist/Sacred Mod Loader.exe`.

Never skip registry or address-table generation. A stale `addr.js` can place a
hook at the wrong address and leave it silent. Address generation may use its
documented download fallback when no `mappings` sibling exists.

## Releases

The publishing workflows use repository-owned versions:

- `coderpack` and `build` read `gradle.properties`.
- `protocol` reads `Cargo.toml`.
- `launcher` reads `.version`.
- `idea` reads `pluginVersion` from `gradle.properties`.
- Each mod has its own version and release tag in the form
  `<id>-v<version>`.

Every repository above except `mods` also has `tools/version.ps1`. Run it with
no argument to print that repository's current version, or with a new one
(`pwsh tools/version.ps1 0.99.1`) to write it everywhere that repository
repeats the number by hand -- a Gradle property, a Cargo manifest, a Javadoc
comment, a README example -- in one pass instead of hunting for each spot.
`mods/tools/version.ps1` is the same idea applied to four mods at once: it
refuses to bump them unless all four already agree, since that is the only
case where "the same number everywhere" still makes sense. None of these
scripts touch `dependencies.json` or a version-catalog entry that pins
*another* repository's release -- raising this repository's own version says
nothing about those, and bumping them is a separate, deliberate step.

On `master`, CI publishes a version only when its release tag does not already
exist, then creates that tag as the release record. A version change does not
ship unless the workflow reaches its publishing step successfully.

For a change that crosses the loader release chain, land and verify
`mappings` first, then release any changed `coderpack` and `protocol`
artifacts. Update `launcher/dependencies.json` to those released versions
before releasing `launcher`. The launcher downloads `coderpack` and `protocol`
independently, so neither release depends on the other unless the change itself
requires coordinated versions.

For a toolchain change, release a changed `coderpack` API before a `build`
release or generated project that names that artifact version. Release `build`
before releasing mods that require its new plugin or linter. Mod releases then
remain independent and use `<id>-v<version>` tags.

`idea` cannot publish until it has been pushed and its CI has run once. Its
workflow is prepared to publish the same plugin zip to the JetBrains
Marketplace and a GitHub release. Once that first push has happened, release
`build` first when the scaffolder changed, then raise `pluginVersion` in
`idea`.

## Rules

- Commit in the repository that owns the changed file. Changes inside a
  submodule belong to that repository’s history and remote. Changes to
  root-owned files such as this guide, the root READMEs, `.gitmodules`, or
  `ancaria.code-workspace` belong to the root repository. `idea` has its own
  repository and remote now. Do not claim or run a release from it until it
  has been pushed and its CI has run once.
- Write the commit subject and nothing else. Imperative, sentence case, no
  full stop, and as short as the change allows -- ten words is the ceiling,
  not the target. Name the one thing that changed: `Register the site
  submodule`, `Add tools/version.ps1`, `Point CLAUDE.md at dependencies.json`,
  `Sync the root German README with the version-update section`. No body, no
  bullet list, and no trailer of any kind, `Co-Authored-By` included.
- Commit as soon as a change is finished, and keep each commit to one change.
  A new file and the edits that start using it are two commits. Files that
  share nothing but the working tree they were found in do not share a commit
  either.
- Push only when I have asked for it. A finished commit is not permission to
  push, and neither is a green build.
- Do not move or rename a project directory. Sibling resolution uses these
  directory names. A move can silently switch a build to downloaded artifacts.
- Do not copy files between repositories to avoid rebuilding. Generate
  `mappings.json`, `agent/src/gen/addr.js`, and
  `launcher/install/payload/` with their owning commands.
- Never type a game address directly into code. Every game address must come
  from a row in `mappings`. A wrong RVA may produce no exception and a hook
  that never fires.
- Read `mappings/CLAUDE.md` before changing any address. Read the owning
  repository’s `CLAUDE.md` before changing its code.

## Gotchas

- If the game runs elevated, the host, launcher, and every attaching probe in
  `research` must also run elevated. A permissions mismatch can look like an
  endless wait for a process the tool can already see.
- `idea` is the only component that consumes another component’s Kotlin
  implementation. Both its New Project dialog and `coderpack new` use
  `dev.ancaria.coderpack:templates`. Do not create a second copy of those
  templates.
- The root repository has no CI. Six component repositories have
  `.github/workflows/build.yml`: `build`, `coderpack`, `protocol`, `launcher`,
  `mods`, and `idea`. `idea`'s workflow cannot run as that project's CI until
  it has been pushed. `mappings` and `research` have no workflow, so run their
  checks manually.
- `launcher` CI deliberately uses the download path without sibling checkouts.
  This tests an isolated launcher clone on every push. Exercise the from-source
  path locally when changing how sibling outputs are built or staged.
- A source build of `launcher` with a `coderpack` sibling does not require a
  `mappings` sibling. `tools/build.ps1` passes `-Mappings` only when that path
  contains `mappings.json`. Otherwise `tools/addr.py` uses its normal fallback
  chain. `-Coderpack none` needs no mappings checkout.
- `launcher/tools/install.ps1` reads the game path from the uncommitted
  `launcher/.local.settings` file. A valid entry looks like
  `sacred=D:\SteamLibrary\steamapps\common\Sacred Gold`. The script is expected
  to throw when this file is absent on a fresh checkout.
