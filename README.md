<div align="center">

![Sacred](https://img.shields.io/badge/Sacred-Community-8B1A1A?style=for-the-badge&labelColor=1C1410)
![Java](https://img.shields.io/badge/Java-21-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white&labelColor=1C1410)
![Rust](https://img.shields.io/badge/Rust-1.98-CE422B?style=for-the-badge&logo=rust&logoColor=white&labelColor=1C1410)
![Go](https://img.shields.io/badge/Go-1.26-00ADD8?style=for-the-badge&logo=go&logoColor=white&labelColor=1C1410)
![License](https://img.shields.io/badge/License-MIT-C9A227?style=for-the-badge&labelColor=1C1410)

</div>

[English](README.EN.md) · [Deutsch](README.DE.md)

# ancaria

Sacred вышла в 2004 году без инструментов для модификации, а сборник Sacred Gold
появился в 2005-м. Теперь моды можно писать на Java через API событий.
Файлы игры на диске при этом не меняются: хуки существуют только в памяти и
исчезают после завершения процесса.

Проект начался с вопроса, который не отпускал меня с детства: можно ли спустя
двадцать лет сделать для 32-битной Windows-игры загрузчик модов в духе Forge?
Можно, но 64-битная JVM не загружается в 32-битный процесс. Поэтому загрузчик
работает в трёх частях. Rust-хост внедряет в игру JavaScript-агент, запускает
рядом JVM и передаёт между ними сообщения строчного протокола. Агент ставит
хуки на инструкции игры, а JVM передаёт события модам. Если событие можно
отменить, игровой поток ждёт ответа мода. Через 250 мс хост отправляет `ok` от
имени мода, который не успел ответить.

Проект предназначен только для одиночной игры. Здесь нет самой Sacred Gold,
средств обхода DRM и функций для мультиплеера. Нужна собственная установленная
копия игры.

В этом репозитории нет кода. Это главная страница
[ancaria.dev](https://ancaria.dev) и рабочая копия с семью git-сабмодулями.

## Компоненты проекта

| Репозиторий | Что внутри |
|---|---|
| [`mappings`](https://github.com/ancaria-dev/mappings) | Реестр адресов для `pureHD.exe` 2.0.2.118: VA, RVA и уровень достоверности каждой записи. |
| [`research`](https://github.com/ancaria-dev/research) | Скрипты дизассемблирования, пробы и заметки о том, как находились адреса и поведение игры. Эти материалы не входят в сборку для игроков. |
| [`coderpack`](https://github.com/ancaria-dev/coderpack) | Frida-агент, Java API для компиляции модов и загрузчик на стороне JVM, который принимает события и передаёт их модам. |
| [`protocol`](https://github.com/ancaria-dev/protocol) | Sacred Communication Protocol и Rust-хост, который передаёт сообщения между игрой и JVM. |
| [`launcher`](https://github.com/ancaria-dev/launcher) | Один исполняемый файл для папки с игрой. В нём можно выбрать моды и запустить Sacred Gold. |
| [`build`](https://github.com/ancaria-dev/build) | Инструменты сборки для авторов модов. Сейчас это Gradle-плагин. Поддержка Maven появится при необходимости. |
| [`mods`](https://github.com/ancaria-dev/mods) | Официальный реестр модов и четыре мода. Лаунчер подключает этот реестр по умолчанию. |
| [`idea`](https://github.com/ancaria-dev/idea) | Плагин для IntelliJ IDEA с мастером нового проекта, конфигурацией запуска Sacred, значками на полях и настройкой папки игры. Репозиторий создан, но пока пуст. |

### Зависимости между репозиториями

```mermaid
graph LR
    mappings --> coderpack
    coderpack -.->|e2e-тест| protocol
    coderpack --> launcher
    protocol --> launcher
    mappings -.->|опционально| launcher
    build --> mods
    coderpack --> mods
    build --> idea
```

Граф без циклов. Пунктир — необязательная или тестовая зависимость. `build`
намеренно не зависит от `coderpack`: линтер и шаблоны используют собственную
заглушку API вместо реальной Maven-зависимости, поэтому граф остаётся
ациклическим даже с учётом того, что CI `coderpack` сверяет номер API-контракта
с исходниками `build` и `launcher` — это чтение исходников для проверки, а не
зависимость публикации. `research` в граф не входит: от него никто не зависит,
и сам он ничего не публикует.

## Игроку

1. Скачайте `Sacred Mod Loader.exe` из
   [релизов](https://github.com/ancaria-dev/launcher/releases).
2. Положите файл в папку Sacred Gold, туда же, где лежит экзешник игры.
3. Запустите, поставьте моды из вкладки Available, нажмите Play.

Устанавливать Java заранее не нужно. Лаунчер ищет JDK 21 или новее сначала в
`launcher\java`, затем в `JAVA_HOME` и в конце в PATH. Если подходящей версии
нет, игра всё равно запускается, но моды не загружаются. Через окно Get Java
можно выбрать JDK и скачать его в `launcher\java` внутри папки игры. PATH,
реестр Windows и каталоги за пределами папки игры не меняются. При первом
запуске рядом с игрой также появляются папки `launcher` и `mods`.

Лаунчер показывает содержимое
[реестра](https://github.com/ancaria-dev/mods) и устанавливает выбранные моды в
папку игры. Другой реестр можно добавить по ссылке для клонирования. Для
приватного репозитория понадобится токен.

## Разработчику

### Свой мод

Командная утилита `coderpack` входит в zip-архив каждого релиза
[build](https://github.com/ancaria-dev/build/releases). Распакуйте архив,
добавьте `bin` в PATH и выполните:

```
coderpack new my-mod
```

В `my-mod/` появится проект с Gradle Wrapper, заполненным блоком `sacred { }` и
готовым слушателем. Команда `gradlew assembleSacredMod` собирает jar для
загрузчика. Пакет, отображаемое имя, автора и другие параметры задают ключами,
перечисленными в `coderpack help`. До первого релиза утилиту нужно собрать из
чекаута `build`: запустите `./gradlew :templates:installDist` в каталоге
`gradle`.

Проект можно создать вручную. Для обычного Gradle-проекта в IntelliJ IDEA
достаточно трёх файлов.

`settings.gradle.kts` задаёт репозитории для плагина и API:

```kotlin
pluginManagement {
    repositories {
        mavenLocal()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositories {
        mavenLocal()
        mavenCentral()
    }
}

rootProject.name = "double-gold"
```

`build.gradle.kts` описывает мод. Плагину обязательны `id` и `entrypoint`.
Параметр `apiVersion` добавляет `dev.ancaria.coderpack:api` как зависимость
`compileOnly`:

```kotlin
plugins {
    id("dev.ancaria.coderpack") version "0.99.0"
}

version = "1.0.0"

sacred {
    id = "double-gold"
    displayName = "Double Gold"
    entrypoint = "demo.DoubleGold"
    apiVersion = "0.99.0"
    author("you")
}
```

Сам мод находится в `src/main/java/demo/DoubleGold.java`:

```java
package demo;

import dev.ancaria.coderpack.api.Context;
import dev.ancaria.coderpack.api.SacredMod;
import dev.ancaria.coderpack.api.event.Gold;

public final class DoubleGold implements SacredMod {

    @Override
    public void onLoad(Context context) {
        context.events().on(Gold.class, e -> {
            if (!e.spending()) e.delta(e.delta() * 2);
        });
    }
}
```

После `gradlew assembleSacredMod` мод удваивает получаемое золото. Метод `on`
регистрирует слушатель-лямбду и возвращает `Handle`, с помощью которого
слушатель можно удалить. Аннотация `@Subscribe` над публичным методом с одним
параметром делает то же самое и поддерживает `priority` и `ignoreCancelled`.
Событие `Gold` приходит до записи значения игрой, поэтому изменённая дельта
попадает в игру. Остальные события находятся в пакете
`dev.ancaria.coderpack.api.event`.

### Сборка

Репозитории рассчитаны на отдельную сборку. `coderpack` скачивает
`mappings.json` на ревизии, указанной в `.mappings-ref`. Для GitHub-релизов
других репозиториев (не Maven-координат) `launcher` и `mods` держат в корне
`dependencies.json` — пин точной версии, а не «последний релиз», так что
чужой плохой релиз не ломает сборку без ведома автора; при наличии
сиблинг-чекаута он всё равно побеждает. `mods` и `idea` получают плагин и API
через Gradle Plugin Portal и Maven Central. Для работы над одним компонентом
клонируйте его репозиторий:

```
git clone https://github.com/ancaria-dev/coderpack.git
```

Семь репозиториев можно получить вместе с этим через
`--recurse-submodules`. Файл `ancaria.code-workspace` открывает папки в одном
окне VS Code. Плагин `idea` пока не опубликован в отдельном репозитории и не
добавлен в `.gitmodules`, поэтому рекурсивный клон его не включает. Совместный
чекаут удобен для разработки, но после публикации зависимостей не требуется
для отдельных сборок.

Результаты сборки:

| Репозиторий | Чем собирается | Что на выходе |
|---|---|---|
| `mappings` | `python mappings.generator.py` | `mappings.json`, из которого остальные компоненты берут адреса |
| `coderpack` | `gradlew build` | `api-0.99.0.jar` для компиляции модов и `zygote-0.99.0.jar` для JVM. CI отдельно упаковывает сгенерированный агент в релизный файл `agent.zip` |
| `protocol` | `cargo build --release` | `target/release/protocol.exe`, хост между игрой и JVM |
| `build` | `./gradlew build` в каталоге `gradle` | Gradle-плагин, линтер и `coderpack-0.99.0.zip` с командной утилитой |
| `launcher` | `pwsh tools/build.ps1` | `dist/Sacred Mod Loader.exe`, около 81 МБ (77 МиБ), со встроенными хостом, jar-файлами и агентом |
| `mods` | `gradlew assembleSacredMod` | Четыре jar-файла, проверенные линтером. `coderpack index` отдельно обновляет `sacred.mods.repository.json` |
| `idea` | `./gradlew buildPlugin` | `build/distributions/sacred-idea-0.99.0.zip`. При выпуске CI также отправляет плагин в JetBrains Marketplace |

У `research` нет общей команды сборки. В репозитории лежат отдельные скрипты
для конкретных исследовательских задач.

Все адреса хуков агент получает из `mappings`. Процесс их поиска описан в
`research`.

В публикуемых репозиториях версия задаётся отдельно. CI выпускает новую сборку,
только если тега `v<версия>` ещё нет, а затем создаёт этот тег. Для выпуска
релиза нужно поднять номер версии. Моды выпускаются по одному с тегами вида
`<id>-v<версия>`.

### Обновление версий

Номер версии живёт не в одном файле. Он размазан по `gradle.properties`,
`Cargo.toml`, README, иногда ещё и по комментарию где-то в коде. Забыть один
из них легко, а потом собранный релиз незаметно расходится с тем, что
написано в документации.

В каждом репозитории, где вообще есть версия, лежит `tools/version.ps1`. Без
аргумента он печатает текущий номер. С аргументом переписывает его сразу
везде, где он встречается в этом репозитории:

```
pwsh tools/version.ps1
pwsh tools/version.ps1 0.99.1
```

## Напоследок

Проект начинался как proof of concept без обещаний долгосрочной поддержки. Его
целью была проверка простой идеи: можно ли написать Java-мод для старой любимой
игры.

## Отсылки

Часть находок в `research` и `mappings` опирается на более раннюю работу
сообщества: реверс-инжиниринг форматов и структур игры в
[SacredGameTools](https://github.com/sonicmouse/SacredGameTools) и
[sacred-sdk](https://github.com/bssth/sacred-sdk). Модификация
[pureHD](https://steamcommunity.com/app/12320/discussions/0/3191364450206457546/)
стала базой для модов — весь реестр адресов в `mappings` собран именно под эту
сборку `pureHD.exe`.

Отдельное спасибо сообществу, которое до сих пор разбирает игру 2004 года,
хотя в современной игровой индустрии таким играм давно предначертана смерть.
Я играл в Path of Exile, Path of Exile 2, Last Epoch и Titan Quest — это
всё не то: ни та атмосфера, ни тот вайб, ни та беззаботная казуальность и
проработанность, что были в Sacred. Таких игр больше никогда не будет, но в
сердце она останется.

## Лицензия

MIT, см. [LICENSE](LICENSE).
