# ancaria workspace

## Working with me

- Chat with me in Russian. Code, commits, and CLAUDE.md files stay in English. READMEs follow the README template below.
- In project prose, refer to me as "I"/"me", never "the author".

## Workspace

- This root repository is the organisation's `.github` repository. It holds workspace files only: guides, READMEs, CONTRIBUTING, `profile/`, `.gitmodules`, `ancaria.code-workspace`, `renovate.json`. No application code.
- The project directories are separate side-by-side clones of `https://github.com/ancaria-dev/<name>.git`, not gitlinks. Never `git add -A` from the root.
- Every repository uses `master`. Each `.gitmodules` entry pins `branch = master`.
- Never move or rename a project directory. Builds resolve siblings by these names, and a move silently switches a build to downloaded artifacts.
- `ancaria.code-workspace` hides the project directories under the root so they do not appear twice.

## Component guides

Read the owning guide before changing its code.

- `mappings/CLAUDE.md`: the address registry. Row format, VA/RVA, confidence levels, export tags, generator, parser traps.
- `research/CLAUDE.md`: disassembly scripts and live probes. Evidence policy, research traps, and "Analysis tools" (Ghidra, capstone, radare2, angr, undname, pymem).
- `coderpack/CLAUDE.md`: Frida agent, Java API, Kotlin API, zygote. Hook safety rules, address generation, API contract, release.
- `protocol/CLAUDE.md`: wire protocol and Rust host. Transports, verdict deadline, embedded agent, host failure modes.
- `launcher/CLAUDE.md`: the Go launcher. Payload, install layout, Java, pureHD, updater, mod install rules, Windows traps.
- `build/CLAUDE.md`: Gradle plugin, mod linter, `coderpack` scaffolder and templates, SRML index.
- `mods/CLAUDE.md`: the default SRML repository and its four mods.
- `idea/CLAUDE.md`: the IntelliJ IDEA plugin.
- `site/CLAUDE.md`: the ancaria.dev front end and its Cloudflare Worker.

## Project facts

- ancaria is a mod loader for Sacred Gold. Mods are Java or Kotlin, written against an event API, loaded while the game runs.
- Every address targets `pureHD.exe` 2.0.2.118, a 32-bit community HD wrapper, image base `0x00400000`. Never assume the stock `Sacred.exe` or any other build uses the same addresses.
- The launcher, host, and agent find the game as `pureHD.exe`, `Sacred.exe`, then `Game.exe`, ignoring case. Detection is not compatibility. The agent's signature-mismatch warning does not make the addresses safe.
- The game is 32-bit and the JVM is 64-bit, so they run as separate processes. The Rust host carries events between them.
- A cancelable event is an `ASK` that stops the game thread while mods decide. Never describe its 250 ms deadline as a strict worst case. Details: `protocol/CLAUDE.md`.
- The loader never patches game files on disk. Hooks live only in the running process.
- The project is single-player only and needs a legally owned installed copy. Never add multiplayer features, DRM bypass, a game executable, or game files.

## Where changes go

- Edit the repository that owns the change: addresses in `mappings`, hooks in `coderpack`, probes in `research`.
- When a probe establishes an address or fact the loader uses, record it in `mappings`.
- Never type a game address into code. Every address comes from a `mappings` row; a wrong RVA can give a hook that never fires and no error. Before changing any address, read `mappings/CLAUDE.md`.
- Never copy files between repositories to skip a rebuild. Generate `mappings.json`, `agent/src/gen/addr.js`, and `launcher/install/payload/` with their owning commands.
- `idea` and `coderpack new` share `dev.ancaria.coderpack:templates`. Never create a second copy of the templates.

## Cross-repository builds

- Each repository builds alone. Its guide says how it falls back from a sibling checkout to a pinned release.
- A repository that needs another repository's GitHub Release asset pins it in a root `dependencies.json`: `[{ "path": "ancaria-dev/<repo>", "version": "<version, no v prefix>" }]`. One Renovate custom manager in `renovate.json` bumps them all, so keep the shape identical.
- Never download "latest". CI requests the exact pinned tag. A sibling checkout always wins over the pin.
- Rebuild a cross-repository change in this order:
  0. Scaffolder or template changed: `cd gradle && ./gradlew publishToMavenLocal` in `build`.
  1. `mappings`: `python mappings.generator.py`, then `python mappings.generator.py --check`.
  2. `coderpack`: `python tools/addr.py`. Also `python tools/hooksafe.py` for every new hooked row.
  3. `coderpack`: `./gradlew build`.
  4. `protocol`: `cargo build --release`. Needs step 2, because the build embeds `gen/addr.js`.
  5. `launcher`: `pwsh tools/build.ps1`.
- Never skip registry or address-table generation. A stale `addr.js` places hooks at wrong addresses and they stay silent.
- `hooksafe.py` reads the installed game, so CI cannot run it. Run it locally for every new hooked row.

## Releases

- `pwsh tools/version.ps1` prints a repository's version. `pwsh tools/version.ps1 <new>` writes it everywhere that repository repeats it. It never touches `dependencies.json` or a pin on another repository; bump those deliberately.
- On `master`, CI publishes only when tag `v<version>` does not exist, then creates it. A version change ships only if the workflow reaches its publish step.
- Loader release chain: land and verify `mappings`; release changed `coderpack`; if the agent changed, raise `protocol/dependencies.json` and release `protocol`; raise `launcher/dependencies.json`; release `launcher`.
- An agent change reaches players only through a `protocol` release, because `protocol.exe` embeds the agent. A workspace build uses the sibling agent and hides a missing release.
- Toolchain chain: release changed `coderpack` API and `api-kotlin` (same version) before a `build` release that names them; release `build` before mods that need its plugin or linter, and before raising `idea`'s `templates` pin.
- A Central upload is not a Central release. `publishingType` is `USER_MANAGED`, so an artifact resolves only after someone presses Publish in the portal. Before pushing a pin on a Central artifact, request its POM.

## Commits

- Commit in the repository that owns the file. Root-owned files (this guide, root READMEs, `.gitmodules`, `ancaria.code-workspace`) go to the root repository.
- Write a subject line only: imperative, sentence case, no full stop, at most ten words, naming the one thing that changed.
- Commit as soon as a change is finished. One change per commit.
- A new file and the edits that start using it are two commits. Unrelated files never share a commit.
- Push only when I ask. A finished commit or a green build is not permission.

## Gotchas

- If the game runs elevated, run the host, launcher, and every attaching probe elevated too. Otherwise the tool waits forever on a process it can see.
- The root repository and `research` have no CI. Every other repository's workflow runs its own tests; `mappings` runs `mappings.generator.py --check`.

# Exact calculation

- Never guess or estimate a number, count, size, offset, address, date,
  duration, percentage, or version comparison. Compute it.
- Compute with a short task-specific Python script you run now. Use the
  exact-calculations skill when it is available. Check which interpreter works
  first: `python --version` (on this machine `python`, not `python3`).
- Money and decimal quantities: `decimal.Decimal` built from strings.
  Exact ratios: `fractions.Fraction`. Never binary float for exact values.
- Dates and durations: `datetime` and `zoneinfo`.
- Hex, RVA/VA, and offsets: compute them in Python (`hex()`, `int(x, 16)`,
  image base 0x00400000). Never add them in your head.
- Counts of files, rows, hooks, or tests: count them with a command. Never
  copy a count from prose.
- Round explicitly, state units and assumptions, and keep the source
  precision.
- If an input is ambiguous, ask me. If execution fails, fix it. Never present
  a number you did not compute.

# Typography

Applies to every text a reader sees: READMEs, the site, launcher strings, release notes, plugin descriptions. Code, identifiers, paths, and commands in backticks keep their literal characters.

## Style in every language

- Lead with the reader. A player wants what they get and how to start. Internals go to the developer section or the owning repository. Each fact lives in one place; others link to it.
- Connect sentences: each follows from the last. No jump from history to a feature without the step between.
- Active voice, strong verbs. Name who acts: the launcher, the host, the mod, you.
- One idea per sentence. Vary sentence length. Two or three sentences per paragraph.
- Cut words that carry no meaning. No filler openings, no summaries of what was just said.
- No bureaucratic noun chains: "we optimise", not "the performing of the optimisation".
- Never use these clichés: delve, tapestry, testament, crucial, beacon, look no further, revolutionize, in conclusion, seamless, robust, leverage; «в современном мире», «динамично развивающийся», «важно отметить»; „In der heutigen, schnelllebigen Welt“, „Es ist wichtig zu betonen“, „Meilenstein“, „einzigartig“, „ganzheitlich“.
- Refer to me in the first person ("I", «я», „ich“), never "the author".
- Semicolons only inside complex lists. Otherwise a full stop or a comma.

## README template

- Every repository keeps `README.md` in Russian, `README.EN.md`, and `README.DE.md`, with the same sections in the same order:
  1. Title and one sentence: what the repository is and who it serves.
  2. Two or three short paragraphs on what the reader gets.
  3. Как начать / Getting started / Erste Schritte: tasks as steps.
  4. Reference sections specific to the repository.
  5. Сборка / Building / Bauen.
  6. Релизы / Releases / Releases.
  7. Благодарности / Acknowledgements / Danksagung, only when there is any.
  8. Лицензия / License / Lizenz.
- Exception: a mod README in `mods/<id>/` is a single English `README.md`.
- Leave out a section with nothing to say.
- Loader-wide build and release steps live in the root `CONTRIBUTING` files. Component READMEs link there, not repeat them.
- Root READMEs address players and mod authors first. `profile/` mirrors them with `../` links.

## English

- Curly quotes: “text”, nested “text ‘inside’ text”.
- Dash: em dash without spaces (text—text) or en dash with spaces (text – text). Never a hyphen as a dash.
- Apostrophe for contractions and possessives: don't, it's, the user's guide.
- Conversational but professional. Address the reader as “you”.

## Russian

- Quotes «ёлочки», nested „лапки“: «слово „слово“ слово».
- Dash: em dash with spaces « — ». Never a hyphen.
- Never an apostrophe in place of ъ.
- Address the reader as «вы», lower case.
- Avoid канцелярит: «осуществить установку» becomes «установить».

## German

- Quotes „Text“: low opening, high closing.
- Dash: en dash with spaces „Text – Text“. Never a hyphen.
- No genitive apostrophe: Peters Auto. Only after s, z, or x: Max' Auto.
- Address the reader as „du“, never „Sie“.
- Avoid Nominalstil. Replace -ung, -heit, -keit nouns with verbs: „Wir optimieren“, not „Die Durchführung der Optimierung“.
- Break up Schachtelsätze. Modal particles (mal, ja, doch, halt) sparingly, only in casual passages.
