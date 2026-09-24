<div align="center">

![Sacred](https://img.shields.io/badge/Sacred-Community-8B1A1A?style=for-the-badge&labelColor=1C1410)
![Java](https://img.shields.io/badge/Java-21-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white&labelColor=1C1410)
![Rust](https://img.shields.io/badge/Rust-1.98-CE422B?style=for-the-badge&logo=rust&logoColor=white&labelColor=1C1410)
![Go](https://img.shields.io/badge/Go-1.26-00ADD8?style=for-the-badge&logo=go&logoColor=white&labelColor=1C1410)
![License](https://img.shields.io/badge/License-MIT-C9A227?style=for-the-badge&labelColor=1C1410)

</div>

[Русский](README.md) · [Deutsch](README.DE.md)

# ancaria

A mod loader for Sacred Gold. You write mods in Java or Kotlin, and the loader
plugs them into the game while it runs.

Sacred came out in 2004, and the Sacred Gold compilation followed in 2005. The
game never got modding tools. ancaria fills that gap: your mod subscribes to
game events, such as picking up gold, and decides what happens next.

Your game files stay untouched. The loader hooks the game only in memory, and
the hooks disappear when the game closes.

The project is for single-player use. It doesn't include the game, bypass DRM,
or add multiplayer features. You need your own installed copy of Sacred Gold.

## Getting started

### Play with mods

1. Download `Sacred Mod Loader.exe` from the
   [releases](https://github.com/ancaria-dev/launcher/releases).
2. Put it in your Sacred Gold folder, next to the game's executable.
3. Run it, pick mods on the Available tab, and press Play.

Mods need Java 21 or newer. If you don't have it, the launcher offers to
download a suitable version into the game folder. It leaves your system alone:
PATH and the Windows registry stay as they were. The
[launcher README](https://github.com/ancaria-dev/launcher) has the details.

### Write a mod

The easiest start is IntelliJ IDEA with the
[Sacred Mod Development](https://plugins.jetbrains.com/plugin/34165-sacred-mod-development)
plugin. It creates a project from File → New → Project → Sacred Mod and starts
the game with your mod through a single Run Sacred button.

Without an IDE, the `coderpack` command from the
[build releases](https://github.com/ancaria-dev/build/releases) creates the
same project:

```
coderpack new my-mod
```

Here's a mod that doubles the gold you pick up:

```java
public final class DoubleGold extends SacredMod {

    @Override
    public void onLoad() {
        getContext().getRegistry().getEventRegistry().register(this);
    }

    @Subscribe
    public Gold.Mutation onGold(Gold event) {
        if (event.isSpending()) {
            return Gold.Mutation.none();
        }
        return Gold.Mutation.change(event.getValue() * 2);
    }
}
```

`gradlew assembleSacredMod` builds a jar that the launcher installs like any
other mod. [coderpack](https://github.com/ancaria-dev/coderpack) explains the
API, lists the events, and shows the Kotlin version.
[build](https://github.com/ancaria-dev/build) covers the build settings.

## How it works

Sacred is a 32-bit game, and the Java that runs mods is 64-bit. They can't
share a process, so the loader has three parts. An agent inside the game
catches its events. A JVM in a separate process hands them to mods. A Rust host
connects the two.

A mod can cancel or change some events. The game then waits for its answer.
If the mod runs late, the host answers “change nothing” on its behalf, usually
after 250–375 ms. Keep those handlers short.

## Repositories

| Repository | What's inside |
|---|---|
| [`launcher`](https://github.com/ancaria-dev/launcher) | The launcher. It installs the loader, manages mods, and starts the game. |
| [`mods`](https://github.com/ancaria-dev/mods) | The official mod repository and the source of four mods. |
| [`coderpack`](https://github.com/ancaria-dev/coderpack) | The Java and Kotlin API for mods, the in-game agent, and the JVM-side mod loader. |
| [`build`](https://github.com/ancaria-dev/build) | The Gradle plugin, the linter, and the `coderpack` command for new projects. |
| [`idea`](https://github.com/ancaria-dev/idea) | The IntelliJ IDEA plugin. |
| [`protocol`](https://github.com/ancaria-dev/protocol) | The Rust host and the protocol between the game and the JVM. |
| [`mappings`](https://github.com/ancaria-dev/mappings) | Addresses of game functions in `pureHD.exe` 2.0.2.118. |
| [`research`](https://github.com/ancaria-dev/research) | The scripts and notes that found those addresses. |
| [`site`](https://github.com/ancaria-dev/site) | The source of [ancaria.dev](https://ancaria.dev). |

This repository holds no code. It joins the others into one workspace.

## Contributing

[CONTRIBUTING.EN.md](../CONTRIBUTING.EN.md) explains how to build the loader, how
the repositories depend on each other, and in what order to release them.

## Acknowledgements

The project grew out of a question I'd carried since childhood: twenty years
on, could Sacred get a Forge-style mod loader? It started as a proof of
concept, and I don't promise long-term support.

Some findings in `research` and `mappings` build on community work.
[SacredGameTools](https://github.com/sonicmouse/SacredGameTools) and
[sacred-sdk](https://github.com/bssth/sacred-sdk) took apart the game's formats
and structures. Every address in `mappings` targets
[pureHD](https://steamcommunity.com/app/12320/discussions/0/3191364450206457546/),
and the mods run on top of that build.

Thanks to everyone still taking this 2004 game apart. I've played Path of
Exile, Path of Exile 2, Last Epoch, and Titan Quest, and none of them is it.
None has the atmosphere, the carefree feel, or the craft Sacred had. Games like
it won't be made again, but this one stays in my heart.

## License

MIT, see [LICENSE](../LICENSE).
