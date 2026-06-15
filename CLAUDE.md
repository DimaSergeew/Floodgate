# CLAUDE.md — Floodgate (форк)

## Что это за проект
**Floodgate** — плагин GeyserMC, позволяющий игрокам Minecraft **Bedrock** (телефон/консоль) заходить на
**Java**-серверы без лицензии Mojang. Работает в паре с **Geyser** (прокси, переводит протокол Bedrock↔Java).
Geyser шлёт на Java-сервер зашифрованный `BedrockData` (xuid, ник, устройство), а Floodgate на стороне сервера
расшифровывает его и решает, какой UUID/ник выдать игроку и подменяет `GameProfile` в процессе логина.

Модули: `core` (общая логика), `api`, `ap` (annotation processor), платформы `spigot`/`bungee`/`velocity`,
БД-привязок `database/*`. Все наши правки — в `core`, поэтому работают на всех платформах.

## Зачем этот форк (наша задача)
Сервер **пиратский (offline-mode, `online-mode=false`)**. Нужно, чтобы игрок заходил **с одного ника и с ПК (Java),
и с телефона (Bedrock)** на **один аккаунт** — общий прогресс, вещи, права, регистрация в плагине авторизации.

Проблема стокового Floodgate: Bedrock-игроку выдавался UUID `00000000-0000-0000-XXXX-(xuid)` и префикс `.` к нику →
для сервера и плагина авторизации это был **отдельный** игрок («1 ник — 2 регистрации»).

## Что мы сделали
Добавили опцию **`use-offline-uuid`**: когда включена, непривязанный Bedrock-игрок получает **тот же offline-UUID,
что и Java-игрок с таким же ником** — `UUID.nameUUIDFromBytes("OfflinePlayer:" + ник)` (формула vanilla offline-mode).
Вместе с пустым `username-prefix` это делает Bedrock- и Java-игрока с одним ником **одним аккаунтом**.

### Изменённые файлы (`core`)
- `util/Utils.java` — добавлен `getOfflineUuid(javaUsername)`.
- `config/FloodgateConfig.java` — поле `useOfflineUuid` (геттер `isUseOfflineUuid()`).
- `resources/config.yml` — `username-prefix: ""`, новая опция `use-offline-uuid: true`.
- `addon/data/HandshakeDataImpl.java` — `javaUniqueId` считается из ника, если опция включена; иначе старое поведение по xuid.
- `player/FloodgatePlayerImpl.java` — переиспользует `handshakeData.getJavaUniqueId()` (единый источник UUID).
- `skin/SkinUploadSocket.java` — **фикс скинов**: приёмник Bedrock-скинов искал игрока по xuid-UUID, что ломалось
  при offline-UUID; добавлен фолбэк-поиск игрока по `xuid` среди подключённых, иначе скины непривязанных игроков не применялись.

### Что НЕ трогали
- `Utils.getJavaUuid(xuid)` — остаётся для xuid-ключа привязок.
- Поиск привязок в `FloodgateHandshakeHandler` (~стр. 268) — по xuid.
- `LinkedPlayer`/`BedrockData` — это типы из артефакта `geyser:common`, в исходниках Floodgate их нет.
- Репозиторий Geyser не менялся — там ничего не требуется.

## Сборка (ВАЖНО для этой машины)
Gradle 7.6 проекта **несовместим** с системным JDK 25 (`JAVA_HOME`) — Kotlin падает с `IllegalArgumentException: 25.0.3`.
Собирать на **JDK 21** и через `shadowJar` (полный `:spigot:build` падает на задаче `delombok` со старым Lombok на JDK 21,
но это не влияет на рабочий jar):

```powershell
$jdk21 = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.11.10-hotspot'
& ".\gradlew.bat" "-Dorg.gradle.java.home=$jdk21" :spigot:shadowJar
& ".\gradlew.bat" "-Dorg.gradle.java.home=$jdk21" :velocity:shadowJar
```
Готовые плагины:
- Spigot/Paper: `spigot/build/libs/floodgate-spigot.jar`
- Velocity:     `velocity/build/libs/floodgate-velocity.jar`

## Настройка на сервере
- `server.properties`: `online-mode=false` (за прокси — `online-mode` ставится на Velocity/Bungee).
- Floodgate `config.yml` (или `proxy-config.yml` на прокси): `username-prefix: ""`, `use-offline-uuid: true`.
- Привязки не нужны → можно `player-link.enabled: false` (убирает лишние запросы к api.geysermc.org при входе).

## Важные нюансы / подводные камни
- **Только offline-mode.** В online UUID выдаёт Mojang, из ника его не вывести.
- **Ник = личность.** Смена геймертега в Xbox → новый ник → новый offline-UUID → для сервера это новый игрок
  (регистрация заново). Старый аккаунт сохраняется; вернёшь старый ник — вернёшься на старый аккаунт.
- **Регистр важен.** offline-UUID чувствителен к точной строке ника.
- **Безопасность.** Вход по нику без проверки → защищает только плагин авторизации (по паролю). Держать **авто-вход
  для Floodgate-игроков выключенным**, иначе Bedrock-клиент с чужим геймертегом может войти в чужой аккаунт без пароля.
- **Старые дубли в БД авторизации.** После перехода почистить старые записи `.Ник` / со старым UUID, оставив чистую.
- **Скины** Bedrock подтягиваются автоматически (через сервис GeyserMC) после входа с небольшой задержкой; привязанные
  игроки получают скин привязанного Java-аккаунта с Mojang.
