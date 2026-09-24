<div align="center">

![Sacred](https://img.shields.io/badge/Sacred-Community-8B1A1A?style=for-the-badge&labelColor=1C1410)
![Java](https://img.shields.io/badge/Java-21-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white&labelColor=1C1410)
![Rust](https://img.shields.io/badge/Rust-1.98-CE422B?style=for-the-badge&logo=rust&logoColor=white&labelColor=1C1410)
![Go](https://img.shields.io/badge/Go-1.26-00ADD8?style=for-the-badge&logo=go&logoColor=white&labelColor=1C1410)
![License](https://img.shields.io/badge/License-MIT-C9A227?style=for-the-badge&labelColor=1C1410)

</div>

[Русский](README.md) · [English](README.EN.md)

# ancaria

Ein Mod-Loader für Sacred Gold. Du schreibst Mods in Java oder Kotlin, und der
Loader hängt sie ins laufende Spiel ein.

Sacred erschien 2004, die Sammlung Sacred Gold folgte 2005. Werkzeuge für Mods
hat das Spiel nie bekommen. ancaria schließt diese Lücke: Dein Mod abonniert
Ereignisse im Spiel, etwa das Aufheben von Gold, und entscheidet, was damit
passiert.

Deine Spieldateien bleiben unangetastet. Der Loader setzt seine Hooks nur im
Arbeitsspeicher, und sie verschwinden, sobald du das Spiel beendest.

Das Projekt ist nur für den Einzelspielermodus gedacht. Es enthält weder das
Spiel noch einen DRM-Bypass noch Mehrspielerfunktionen. Du brauchst deine
eigene installierte Kopie von Sacred Gold.

## Erste Schritte

### Mit Mods spielen

1. Lade `Sacred Mod Loader.exe` aus den
   [Releases](https://github.com/ancaria-dev/launcher/releases) herunter.
2. Leg die Datei in deinen Sacred-Gold-Ordner, neben die EXE des Spiels.
3. Starte sie, wähl im Tab Available deine Mods aus und drück auf Play.

Mods brauchen Java 21 oder neuer. Fehlt es, bietet der Launcher an, eine
passende Version in den Spielordner zu laden. Dein System lässt er in Ruhe:
PATH und die Windows-Registry bleiben, wie sie sind. Details stehen in der
README des [Launchers](https://github.com/ancaria-dev/launcher).

### Einen Mod schreiben

Am einfachsten startest du in IntelliJ IDEA mit dem Plugin
[Sacred Mod Development](https://plugins.jetbrains.com/plugin/34165-sacred-mod-development).
Es legt über File → New → Project → Sacred Mod ein Projekt an und startet das
Spiel mit deinem Mod per Klick auf Run Sacred.

Ohne IDE legt das Werkzeug `coderpack` aus den
[Releases von build](https://github.com/ancaria-dev/build/releases) dasselbe
Projekt an:

```
coderpack new my-mod
```

Dieser Mod verdoppelt das Gold, das du aufhebst:

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

`gradlew assembleSacredMod` baut ein JAR, das der Launcher wie jeden anderen
Mod installiert. [coderpack](https://github.com/ancaria-dev/coderpack) erklärt
die API, listet die Ereignisse auf und zeigt die Kotlin-Variante.
[build](https://github.com/ancaria-dev/build) beschreibt die
Build-Einstellungen.

## So funktioniert es

Sacred ist ein 32-Bit-Spiel, das Java für die Mods läuft aber mit 64 Bit. In
einem Prozess vertragen sie sich nicht, deshalb besteht der Loader aus drei
Teilen. Ein Agent im Spiel fängt dessen Ereignisse ab. Eine JVM in einem
eigenen Prozess reicht sie an die Mods weiter. Ein Host in Rust verbindet
beide Seiten.

Manche Ereignisse kann ein Mod abbrechen oder ändern. Dann wartet das Spiel auf
seine Antwort. Ist der Mod zu langsam, antwortet der Host für ihn mit „nichts
ändern“, meist nach 250–375 ms. Halt solche Handler also kurz.

## Repositories

| Repository | Was drin ist |
|---|---|
| [`launcher`](https://github.com/ancaria-dev/launcher) | Der Launcher. Er installiert den Loader, verwaltet Mods und startet das Spiel. |
| [`mods`](https://github.com/ancaria-dev/mods) | Das offizielle Mod-Repository und die Quellen von vier Mods. |
| [`coderpack`](https://github.com/ancaria-dev/coderpack) | Die API für Mods in Java und Kotlin, der Agent im Spiel und der Mod-Loader auf der JVM-Seite. |
| [`build`](https://github.com/ancaria-dev/build) | Das Gradle-Plugin, der Linter und das Werkzeug `coderpack` für neue Projekte. |
| [`idea`](https://github.com/ancaria-dev/idea) | Das Plugin für IntelliJ IDEA. |
| [`protocol`](https://github.com/ancaria-dev/protocol) | Der Host in Rust und das Protokoll zwischen Spiel und JVM. |
| [`mappings`](https://github.com/ancaria-dev/mappings) | Adressen der Spielfunktionen in `pureHD.exe` 2.0.2.118. |
| [`research`](https://github.com/ancaria-dev/research) | Die Skripte und Notizen, mit denen diese Adressen gefunden wurden. |
| [`site`](https://github.com/ancaria-dev/site) | Die Quellen von [ancaria.dev](https://ancaria.dev). |

Dieses Repository enthält keinen Code. Es fasst die übrigen zu einem
Arbeitsbereich zusammen.

## Mitentwickeln

Wie du den Loader baust, wie die Repositories voneinander abhängen und in
welcher Reihenfolge Releases erscheinen, steht in
[CONTRIBUTING.DE.md](../CONTRIBUTING.DE.md).

## Danksagung

Das Projekt ist aus einer Frage entstanden, die mich seit meiner Kindheit
begleitet: Bekommt Sacred nach zwanzig Jahren doch noch einen Mod-Loader wie
Forge? Es begann als Proof of Concept, und eine langfristige Pflege verspreche
ich nicht.

Einige Erkenntnisse in `research` und `mappings` bauen auf der Arbeit der
Community auf. [SacredGameTools](https://github.com/sonicmouse/SacredGameTools)
und [sacred-sdk](https://github.com/bssth/sacred-sdk) haben die Formate und
Strukturen des Spiels zerlegt. Alle Adressen in `mappings` beziehen sich auf
[pureHD](https://steamcommunity.com/app/12320/discussions/0/3191364450206457546/),
und die Mods laufen auf diesem Build.

Danke an alle, die dieses Spiel von 2004 bis heute auseinandernehmen. Ich habe
Path of Exile, Path of Exile 2, Last Epoch und Titan Quest gespielt, aber das
ist halt nicht dasselbe. Keinem davon gelingt diese Atmosphäre, diese
Unbeschwertheit und diese Liebe zum Detail wie Sacred. Solche Spiele wird es
nicht mehr geben, im Herzen bleibt dieses aber.

## Lizenz

MIT, siehe [LICENSE](../LICENSE).
