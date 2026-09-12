#German

# PC_Overlay
Just a Small (AI Written/Forgive me) PC / Server usable Overlay, when u want to for example want to Overlook the Ressources or media , u can also Interact with data on ur Server per SSH  send and recieve files (rn rudimentary/ can and might be improved over time)

# PC Overlay

Kleine Web-Oberfläche zum Managen eines Linux-Servers oder einer Workstation
im lokalen Netz. Einzelne Python-Datei, keine externen Dienste.

## Features

- Systemübersicht (CPU, RAM, Load, Uptime, Disks)
- Speichermedien-Auswahl (Auto-Filter < 5 GiB, manuell anpassbar)
- Prozessliste mit Kill
- systemd-Dienste starten/stoppen/restarten, Journal-Logs
- Datei-Browser mit Upload, Download, ZIP-Export, Editor
- Shell (optional per Whitelist einschränkbar)
- Mehrere SSH-Ziele als zusätzliche „Overlays" verwaltbar

## Installation

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install fastapi uvicorn paramiko psutil
python3 pcoverlay.py
```

Beim ersten Start wirst du nach Bind-IP, Port, Benutzer und Passwort gefragt.
Die Konfiguration landet in `config.json` (chmod 600, nicht committen).

## Sicherheitshinweise

- Nur für den Betrieb im lokalen Netz oder hinter VPN gedacht.
- HTTP Basic Auth ohne TLS. Für externen Zugriff Reverse-Proxy mit TLS nutzen.
- SSH- und Sudo-Passwörter liegen im Klartext in `config.json`.
- Der Shell-Endpunkt ist standardmäßig ungefiltert. `shell_whitelist` in der
  Konfiguration setzen, um Befehlspräfixe einzuschränken.

#English

# PC Overlay

A lightweight web interface for managing a Linux server or workstation
on a local network. Single Python file, no external services.

## Features

- System overview (CPU, RAM, load, uptime, disks)
- Storage media selection (auto-filter < 5 GiB, manually adjustable)
- Process list with kill capability
- Start/stop/restart systemd services, view journal logs
- File browser with upload, download, ZIP export, and editor
- Shell (optionally restrictable via whitelist)
- Manage multiple SSH targets as additional "overlays"

## Installation

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install fastapi uvicorn paramiko psutil
python3 pcoverlay.py
```

Upon first launch, you will be prompted for the bind IP, port, username, and password.
The configuration is saved to `config.json` (chmod 600; do not commit to version control).

## Security Notes

- Intended for use on a local network or behind a VPN only.
- Uses HTTP Basic Auth without TLS; use a reverse proxy with TLS for external access.
- SSH and sudo passwords are stored in plain text in `config.json`.
- The shell endpoint is unfiltered by default; set `shell_whitelist` in the
configuration to restrict command prefixes.

#Русский

# PC Overlay

Небольшой веб-интерфейс для управления сервером или рабочей станцией Linux
в локальной сети. Один файл Python, не требуется никаких внешних служб.

## Возможности

- Обзор системы (ЦП, ОЗУ, загрузка, время работы, диски)

- Выбор носителя данных (автоматический фильтр < 5 ГБ, настраиваемый вручную)

- Список процессов с функцией завершения
- Запуск/остановка/перезапуск служб systemd, журналы событий

- Файловый браузер с загрузкой, скачиванием, экспортом в ZIP-архив, редактор

- Оболочка (опционально с возможностью добавления в белый список)

- Возможность управления несколькими SSH-целями в качестве дополнительных «оверлеев»

## Установка

```bash

python3 -m venv .venv

source .venv/bin/activate

pip install fastapi uvicorn paramiko psutil
python3 pcoverlay.py
```

При первом запуске вам будет предложено ввести IP-адрес привязки, порт, имя пользователя и пароль.

Конфигурация хранится в файле `config.json` (chmod 600, не фиксировать изменения).

## Примечания по безопасности

- Предназначено для использования только в локальной сети или за VPN.

- Базовая HTTP-аутентификация без TLS. Для внешнего доступа используйте обратный прокси с TLS.

- Пароли SSH и sudo хранятся в открытом виде в файле `config.json`.

- Конечная точка оболочки по умолчанию не фильтруется. Установите `shell_whitelist` в конфигурации, чтобы ограничить префиксы команд.

#日本語

