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
