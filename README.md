<div align="center">

# ⬢ MESHWDTT

**Личный VPN-канал через TURN-инфраструктуру звонков VK**

Android-клиент + серверная часть с собственным управлением паролями.

`Android 9+` · `Go 1.25` · `Jetpack Compose` · `GPL v3`

</div>

---

## Как это работает

MESHWDTT создаёт на телефоне системный VPN-туннель и передаёт зашифрованный трафик
через TURN-серверы, используемые для звонков VK. Выход в интернет происходит через ваш
собственный VPS. Для посторонней наблюдательной сети такое соединение выглядит как
обычный медиатрафик WebRTC-звонка, а не как прямое подключение к VPN-серверу.

```
Телефон ── WireGuard/RAW туннель ──► TURN (VK) ──► Ваш VPS ──► Интернет
```

### Возможности

- 🛡 **Два режима туннеля** — WireGuard поверх TURN или RAW-режим (userspace tun).
- 📱 **Управление профилями** — создание, импорт по ссылке/QR, подписки, экспорт.
- 🔗 **Deep-link импорт** — ссылки вида `meshwdtt://config?...`, файлы `.meshwdtt` и `.conf`.
- 🖥 **Быстрые переключатели** — виджет на домашний экран и плитка в шторке.
- 🧩 **Обход блокировок** — списки доменов/IP и исключения приложений, шаринг настроек.
- 🤖 **Telegram-бот администратора** — выпуск и отзыв временных паролей прямо на сервере.
- 🔄 **Автообновления** — проверка новых версий через GitHub Releases.
- 🌐 **DNS через DoH**, генерация реалистичного User-Agent для VK-руки.

---

## Структура проекта

| Каталог | Назначение |
|---|---|
| `app/` | Android-приложение (Kotlin + Jetpack Compose) |
| `go_client/` | Нативное ядро клиента на Go → собирается в `libclient.so` |
| `server/` | Серверная часть на Go + Telegram-бот управления паролями |
| `scripts/` | Сборка нативных библиотек под ABI Android |
| `.github/workflows/` | CI: debug-сборки и подписанные релизы |

---

## 🚀 Как собрать APK

### Быстрый способ — GitHub Actions (локальная установка не нужна)

Репозиторий уже содержит готовые workflow-файлы:

1. Запушьте код в свой GitHub-репозиторий.
2. **Debug-сборка:** отправьте PR в `develop` или `master`, либо вручную запустите
   workflow **«Android debug APK»** (Actions → *Android debug APK* → *Run workflow*).
   Через несколько минут в конце запуска появится артефакт **`app-arm64-v8a-debug.apk`**.
3. **Release-сборка:** добавьте секреты репозитория
   (`ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`,
   `ANDROID_KEY_PASSWORD`) и создайте тег, равный `versionName`:
   ```bash
   git tag v1.5.0 && git push origin v1.5.0
   ```
   Workflow соберёт подписанные APK для трёх ABI, проверит подпись `apksigner`
   и опубликует их в GitHub Releases файлами `meshwdtt-1.5.0-arm64-v8a.apk` и т.д.

> ⚠️ Имя тега **должно** совпадать со значением `versionName` в `app/build.gradle.kts`,
> иначе CI отклонит сборку.

### Локальная сборка

**Требуется установить:**

| Инструмент | Версия | Зачем |
|---|---|---|
| JDK | 17 | Gradle / Android Gradle Plugin |
| Android SDK | Platform **android-35**, Build-Tools **35.0.0** | компиляция приложения |
| Android NDK | **27.2.12479018** | кросс-компиляция Go-ядра под ARM |
| Go | 1.25+ | сборка `libclient.so` и серверного бинарника |
| Gradle | не нужен | используется wrapper (`./gradlew`, Gradle 9.1.0) |

**Шаг 1. Настройка окружения (Linux/macOS):**

```bash
export ANDROID_HOME="$HOME/Android/Sdk"        # путь к Android SDK
export PATH="$ANDROID_HOME/platform-tools:$PATH"
sdkmanager "platforms;android-35" "build-tools;35.0.0" "ndk;27.2.12479018"
```

Для Windows используйте Android Studio (SDK Manager → тот же набор),
а команды запускайте из `cmd`/PowerShell без `./`.

**Шаг 2. Клонирование:**

```bash
git clone https://github.com/<ВАШ_АККАУНТ>/meshwdtt-android.git
cd meshwdtt-android
```

**Шаг 3. Сборка debug-APK:**

```bash
./gradlew :app:assembleDebug -PtargetAbis=arm64-v8a --no-daemon
```

Готовый файл: `app/build/outputs/apk/debug/app-arm64-v8a-debug.apk`

**Шаг 4. Сборка release-APK (без подписи — подпись debug-ключом):**

```bash
./gradlew :app:assembleRelease -PtargetAbis=arm64-v8a,armeabi-v7a,x86_64 --no-daemon
```

APK появятся в `app/build/outputs/apk/release/` (по одному на ABI + universal).

**Шаг 5. Подпись своим ключом (опционально, но нужно для автообновлений и магазина):**

```bash
keytool -genkeypair -v -keystore release.keystore -alias meshwdtt \
  -keyalg RSA -keysize 2048 -validity 10000

# в local.properties (в корне проекта) добавить:
KEYSTORE_FILE=../release.keystore
KEYSTORE_PASSWORD=ваш_пароль
KEY_ALIAS=meshwdtt
KEY_PASSWORD=ваш_пароль
```

После этого `./gradlew :app:assembleRelease` автоматически использует этот ключ.

### Что происходит при сборке

Задача `preBuild` автоматически запускает две нативные сборки:

1. **`buildNativeLibs`** → `scripts/build-native-libs.sh` → `scripts/build-go-lib.sh`
   кросс-компилирует `go_client/` через NDK clang в
   `app/src/main/jniLibs/<abi>/libclient.so` (архивы Go-кода, WireGuard, TURN, DoH).
2. **`buildServerAsset`** собирает `server/` в Linux-бинарник и кладёт его в
   `app/src/main/assets/server` — приложение использует его для установки
   серверной части на VPS по SSH.

Поэтому **без установленного Go и NDK сборка не пройдёт** — это частая причина ошибок.

### Типичные проблемы

| Ошибка | Решение |
|---|---|
| `NDK not found` | установите NDK `27.2.12479018` и задайте `ANDROID_NDK_HOME` / `ANDROID_HOME` |
| `invalid go version '1.2x.x'` | обновите локальный Go до 1.25+ (старые Go не понимают новые `go.mod`) |
| `unsupported ldflags: -checklinkname=0` | у вас Go ≥ 1.23 — скрипт добавляет флаг сам; обновите `scripts/` |
| Нет `libclient.so` в APK | проверьте, что `jniLibs/<abi>/libclient.so` создан до сборки Gradle |
| Release не устанавливается поверх старого | сменился `applicationId` (`net.meshwdtt.client`) — удалите старую версию |

---

## Установка сервера

Сервер ставится на чистый Ubuntu/Debian VPS одним из способов:

**Через приложение (рекомендуется).** Вкладка **«Серверы»** → добавить хост, SSH-логин
и пароль/ключ → «Установить». Приложение само скопирует бинарник `server` и `deploy.sh`
(из assets) и выполнит настройку.

**Вручную.** Соберите сервер и загрузите на VPS:

```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -trimpath -o server-bin ./server
scp server-bin deploy.sh user@vps:/opt/
ssh user@vps 'sudo bash /opt/deploy.sh'
```

После установки управление паролями ведётся через Telegram-бота:

```
/new    — создать временный пароль (срок, лимит устройств, VK-хеш)
/list   — список активных паролей
/start  — справка бота
```

Ссылки на профили бот отдаёт в двух видах: `meshwdtt://config?...` (открывается в приложении)
и legacy `wdtt://ip:port:port:port:pass:hash`.

---

## Подключение клиента

1. Установите APK и откройте приложение.
2. Вкладка **«Профили»** → «Добавить профиль» (или импортируйте ссылку/QR/подписку).
3. Укажите адрес сервера, VK-хеш звонка и пароль; при необходимости проверьте хеши.
4. Нажмите **«Подключить»** и подтвердите запрос Android на создание VPN.
5. Контроль состояния — в уведомлении, виджете или плитке шторки.

Формат deep-link:

```
meshwdtt://config?name=Имя&peer=IP&hashes=vk_hash1,vk_hash2&workers=9&port=9000&pass=пароль
```

Приложение также принимает легаси-схемы `qwdtt://` и `wdtt://`, а файлы обхода блокировок
распознаются в новом формате `meshwdtt-bypass` и в старом `qwdtt-bypass`.

---

## Разработка

```bash
./gradlew :app:assembleDebug          # debug APK
./gradlew :app:lint                   # линтер Android
go vet ./server/...                   # проверка серверного кода
go build ./server                     # серверный бинарник локально
```

Ключевые точки кода:

- `app/src/main/java/com/meshwdtt/client/tunnel/TunnelService.kt` — жизненный цикл туннеля.
- `app/src/main/java/com/meshwdtt/client/ui/` — экраны (Compose): профили, серверы, настройки, обход.
- `app/src/main/java/com/meshwdtt/client/profiles/SubscriptionImport.kt` — разбор ссылок/JSON.
- `go_client/` — протокол, TURN-воркеры, капча, SOCKS5, статистика.
- `server/database_bot.go` — Telegram-бот и выдача паролей.

---

## Лицензия

Проект распространяется по лицензии **[GNU GPL v3](LICENSE)**.

Кодовая база развилась из оригинального открытого проекта WDTT; MESHWDTT является
самостоятельным развитием этой базы и не связан с оригинальными авторами.
