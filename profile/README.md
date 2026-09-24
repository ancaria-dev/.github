<div align="center">

![Sacred](https://img.shields.io/badge/Sacred-Community-8B1A1A?style=for-the-badge&labelColor=1C1410)
![Java](https://img.shields.io/badge/Java-21-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white&labelColor=1C1410)
![Rust](https://img.shields.io/badge/Rust-1.98-CE422B?style=for-the-badge&logo=rust&logoColor=white&labelColor=1C1410)
![Go](https://img.shields.io/badge/Go-1.26-00ADD8?style=for-the-badge&logo=go&logoColor=white&labelColor=1C1410)
![License](https://img.shields.io/badge/License-MIT-C9A227?style=for-the-badge&labelColor=1C1410)

</div>

[English](README.EN.md) · [Deutsch](README.DE.md)

# ancaria

Загрузчик модов для Sacred Gold. Моды пишутся на Java или Kotlin и
подключаются к игре прямо во время её работы.

Sacred вышла в 2004 году, сборник Sacred Gold — в 2005-м. Инструментов для
модов у игры так и не появилось. ancaria восполняет этот пробел: вы
подписываетесь на события игры, например на получение золота, и решаете, что
с ними делать.

Файлы игры на диске остаются нетронутыми. Загрузчик ставит хуки только в
памяти, и они исчезают вместе с процессом игры.

Проект работает только в одиночной игре. В нём нет самой игры, обхода DRM и
функций для мультиплеера. Нужна своя установленная копия Sacred Gold.

## Как начать

### Играть с модами

1. Скачайте `Sacred Mod Loader.exe` из
   [релизов](https://github.com/ancaria-dev/launcher/releases).
2. Положите его в папку Sacred Gold, рядом с исполняемым файлом игры.
3. Запустите, выберите моды во вкладке Available и нажмите Play.

Для модов нужна Java 21 или новее. Если её нет, лаунчер предложит скачать
подходящую версию в папку игры. Систему он не трогает: PATH и реестр Windows
остаются как были. Подробности — в README
[лаунчера](https://github.com/ancaria-dev/launcher).

### Написать мод

Проще всего начать в IntelliJ IDEA с плагином
[Sacred Mod Development](https://plugins.jetbrains.com/plugin/34165-sacred-mod-development).
Он создаёт проект через File → New → Project → Sacred Mod и запускает игру с
вашим модом одной кнопкой Run Sacred.

Без IDE проект создаёт утилита `coderpack` из
[релизов build](https://github.com/ancaria-dev/build/releases):

```
coderpack new my-mod
```

Вот мод, который удваивает получаемое золото:

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

`gradlew assembleSacredMod` собирает jar, который лаунчер ставит как любой
другой мод. Как устроен API, какие есть события и как писать на Kotlin,
описано в [coderpack](https://github.com/ancaria-dev/coderpack). Настройки
сборки мода — в [build](https://github.com/ancaria-dev/build).

## Как это устроено

Sacred — 32-битная игра, а Java для модов 64-битная. Внутри одного процесса
они не уживаются, поэтому загрузчик состоит из трёх частей. Агент внутри игры
ловит её события. JVM в отдельном процессе передаёт их модам. Хост на Rust
соединяет обе стороны.

Некоторые события мод может отменить или изменить. Тогда игра ждёт его ответа.
Если мод не успел, хост отвечает за него «ничего не менять», обычно через
250–375 мс. Поэтому такие обработчики должны быть короткими.

## Репозитории

| Репозиторий | Что внутри |
|---|---|
| [`launcher`](https://github.com/ancaria-dev/launcher) | Лаунчер: ставит загрузчик, управляет модами и запускает игру. |
| [`mods`](https://github.com/ancaria-dev/mods) | Официальный реестр модов и исходники четырёх модов. |
| [`coderpack`](https://github.com/ancaria-dev/coderpack) | API для модов на Java и Kotlin, агент внутри игры и загрузчик модов на стороне JVM. |
| [`build`](https://github.com/ancaria-dev/build) | Gradle-плагин, линтер и утилита `coderpack` для новых проектов. |
| [`idea`](https://github.com/ancaria-dev/idea) | Плагин для IntelliJ IDEA. |
| [`protocol`](https://github.com/ancaria-dev/protocol) | Хост на Rust и протокол между игрой и JVM. |
| [`mappings`](https://github.com/ancaria-dev/mappings) | Адреса функций игры в `pureHD.exe` 2.0.2.118. |
| [`research`](https://github.com/ancaria-dev/research) | Скрипты и заметки, с помощью которых эти адреса нашли. |
| [`site`](https://github.com/ancaria-dev/site) | Исходники [ancaria.dev](https://ancaria.dev). |

Этот репозиторий кода не содержит. Он собирает остальные в одну рабочую копию.

## Участие в разработке

Как собрать загрузчик, как репозитории зависят друг от друга и в каком порядке
выпускать релизы, описано в [CONTRIBUTING.md](../CONTRIBUTING.md).

## Благодарности

Проект вырос из вопроса, который не отпускал меня с детства: можно ли спустя
двадцать лет сделать для Sacred загрузчик модов в духе Forge? Он начинался как
proof of concept, и долгосрочной поддержки я не обещаю.

Часть находок в `research` и `mappings` опирается на работу сообщества:
[SacredGameTools](https://github.com/sonicmouse/SacredGameTools) и
[sacred-sdk](https://github.com/bssth/sacred-sdk) разобрали форматы и структуры
игры. Все адреса в `mappings` относятся к
[pureHD](https://steamcommunity.com/app/12320/discussions/0/3191364450206457546/),
и моды работают поверх этой сборки.

Спасибо всем, кто до сих пор разбирает игру 2004 года. Я играл в Path of Exile,
Path of Exile 2, Last Epoch и Titan Quest, но это всё не то. Ни у одной из них
нет той атмосферы, беззаботности и проработки, что были у Sacred. Таких игр
больше не будет, но эта останется в сердце.

## Лицензия

MIT, см. [LICENSE](../LICENSE).
