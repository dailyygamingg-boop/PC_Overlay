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

PCオーバーレイ

PC/サーバーで使用できる、小規模な（AIで作成したため、お粗末な出来栄えですが）オーバーレイです。例えば、リソースやメディアを管理したり、SSH経由でサーバー上のデータにアクセスしてファイルの送受信を行ったりできます（現在は基本的な機能しか備えていませんが、今後改善していく予定です）。

PCオーバーレイ

ローカルネットワーク上のLinuxサーバーまたはワークステーションを管理するための、小規模なWebインターフェースです。単一のPythonファイルで構成されており、外部サービスは不要です。


... 機能

システム概要（CPU、RAM、負荷、稼働時間、ディスク）

ストレージメディアの選択（5 GiB未満で自動フィルタリング、手動調整可能）

プロセス一覧と強制終了機能

systemdサービスの開始/停止/再起動、ジャーナルログ

ファイルブラウザ（アップロード、ダウンロード、ZIPエクスポート、エディタ機能付き）

シェル（オプションでホワイトリスト登録可能）

複数のSSH接続先をオーバーレイとして管理

インストール

```bash

python3 -m venv .venv

source .venv/bin/activate

pip install fastapi uvicorn paramiko psutil

python3 pcoverlay.py

```

初回起動時に、バインドIPアドレス、ポート番号、ユーザー名、パスワードの入力を求められます。設定はconfig.jsonに保存されます（chmod 600、コミットしないでください）。

セキュリティに関する注意事項

ローカルネットワークまたはVPN接続環境でのみ使用することを想定しています。

TLSを使用しないHTTP基本認証です。外部アクセスには、TLS対応のリバースプロキシを使用してください。 SSHとsudoのパスワードはconfig.jsonに平文で保存されます。

シェルエンドポイントはデフォルトではフィルタリングされていません。コマンドプレフィックスを制限するには、設定ファイルで`shell_whitelist`を設定してください。


#한국어

# PC_Overlay
간단한 PC/서버용 오버레이입니다(AI로 작성되었으니 양해 부탁드립니다). 예를 들어, 리소스나 미디어를 관리할 수 있고, SSH를 통해 서버의 데이터와 상호 작용하여 파일을 주고받을 수도 있습니다(현재는 기본적인 기능만 제공하며, 향후 개선될 예정입니다).

# PC Overlay

로컬 네트워크에서 Linux 서버 또는 워크스테이션을 관리하기 위한 간단한 웹 인터페이스입니다.
단일 Python 파일로 구현되었으며, 외부 서비스는 사용하지 않습니다.

`` ## 기능

- 시스템 개요 (CPU, RAM, 로드, 가동 시간, 디스크)

- 저장 매체 선택 (자동 필터 < 5GiB, 수동 조정 가능)

- 프로세스 목록 및 종료 기능

- systemd 서비스 시작/중지/재시작, 저널 로그

- 파일 탐색기 (업로드, 다운로드, ZIP 내보내기, 편집기 포함)

- 셸 (화이트리스트를 통해 접근 제한 가능)

- 여러 SSH 대상을 추가 오버레이로 관리 가능

## 설치

```bash

python3 -m venv .venv

source .venv/bin/activate

pip install fastapi uvicorn paramiko psutil
python3 pcoverlay.py

```

처음 실행 시 바인딩할 IP 주소, 포트, 사용자 이름, 비밀번호를 입력하라는 메시지가 표시됩니다.

설정은 `config.json` 파일에 저장됩니다(권한 600으로 설정, 커밋하지 마십시오).

``bash
` ... ## 보안 참고 사항

- 로컬 네트워크 또는 VPN 환경에서만 사용하도록 설계되었습니다.

- TLS를 사용하지 않는 HTTP 기본 인증 방식입니다. 외부에서 접속하려면 TLS를 지원하는 리버스 프록시를 사용하십시오.

- SSH 및 sudo 암호는 `config.json` 파일에 평문으로 저장됩니다.

- 셸 엔드포인트는 기본적으로 필터링되지 않습니다. 명령 접두사를 제한하려면 구성 파일에서 `shell_whitelist`를 설정하십시오.


#中国

# PC_Overlay

一个小型（AI编写/请见谅）可在PC/服务器上使用的界面。例如，您可以监控资源或媒体，还可以通过SSH与服务器上的数据交互，发送和接收文件（目前功能较为基础，未来会不断改进）。

# PC Overlay

用于管理本地网络上的Linux服务器或工作站的小型Web界面。

仅使用一个Python文件，无需外部服务。

`` ## 功能

- 系统概览（CPU、内存、负载、运行时间、磁盘）

- 存储介质选择（自动过滤小于 5 GiB，可手动调节）

- 进程列表（带终止功能）

- 启动/停止/重启 systemd 服务，日志记录

- 文件浏览器（支持上传、下载、ZIP 导出和编辑）

- Shell（可选白名单限制）

- 可管理多个 SSH 目标，作为额外的覆盖层

## 安装

```bash

python3 -m venv .venv

source .venv/bin/activate

pip install fastapi uvicorn paramiko psutil

python3 pcoverlay.py

```

首次启动时，系统会提示您输入绑定 IP 地址、端口、用户名和密码。

配置信息保存在 `config.json` 文件中（权限设置为 600，请勿提交）。

``bash

` ... ## 安全注意事项

- 仅限在本地网络或 VPN 后使用。

- 使用 HTTP 基本身份验证，不使用 TLS。如需外部访问，请使用启用 TLS 的反向代理。

- SSH 和 sudo 密码以明文形式存储在 `config.json` 文件中。

- shell 端点默认未过滤。请在配置中设置 `shell_whitelist` 以限制命令前缀。
