#Just download this file,and erase this line before you want to start this Webinterface script


#!/usr/bin/env python3
"""
PC Overlay - Web-Oberfläche zum Managen eines Servers.

Nur WebUI-Modus. Öffne die angezeigte URL im Browser.

Aufrufe:
  python3 pcoverlay.py
  python3 pcoverlay.py --reset
  python3 pcoverlay.py --config FILE
  python3 pcoverlay.py --lang de|en|ru|ja|ko|zh

Sicherheit / Deployment:
  - Standardmäßig bindet der Dienst an 0.0.0.0. Zugriff ist per Default auf
    private Netze (RFC1918, Loopback, ULA) beschränkt.
  - Login ist HTTP Basic. Für Zugriff über unsichere Netze einen Reverse-Proxy
    mit TLS davorschalten (nginx, caddy, traefik, ...).
  - SSH- und Sudo-Passwörter liegen im Klartext in config.json. Die Datei
    wird beim Speichern auf chmod 600 gesetzt - trotzdem: nicht committen.
  - Shell-Endpunkt führt beliebige Befehle aus, sofern keine
    shell_whitelist in config.json gesetzt ist.

Dateien:
  config.json   Laufzeit-Konfiguration (nicht versionieren, enthält Secrets)
  .gitignore    sollte config.json ausschließen

Abhängigkeiten:
  pip install fastapi uvicorn paramiko psutil
"""

from __future__ import annotations

import argparse
import hashlib
import io
import ipaddress
import json
import os
import platform
import secrets
import shlex
import shutil
import socket
import subprocess
import sys
import threading
import time
import uuid
import zipfile
from datetime import datetime
from pathlib import Path

import paramiko
import psutil
import uvicorn
from fastapi import Depends, FastAPI, File, Form, HTTPException, Request, UploadFile
from fastapi.responses import FileResponse, HTMLResponse, JSONResponse
from fastapi.security import HTTPBasic, HTTPBasicCredentials

# =============================================================
# KONSTANTEN
# =============================================================
APP_NAME = "PC Overlay"
DISK_MIN_AUTO_BYTES = 5 * 1024 ** 3
SUPPORTED_LANGS = ("de", "en", "ru", "ja", "ko", "zh")
DEFAULT_LANG = "en"

FORBIDDEN_PORTS = {
    20, 21, 22, 23, 25, 53, 67, 68, 69, 80, 110, 111, 123, 135, 137, 138, 139,
    143, 161, 162, 179, 389, 443, 445, 465, 500, 514, 515, 520, 587, 623, 631,
    636, 873, 902, 989, 990, 993, 995, 1080, 1194, 1433, 1521, 1723, 1883,
    2049, 2082, 2083, 2086, 2087, 2095, 2096, 2181, 2222, 2375, 2376, 3000,
    3128, 3260, 3306, 3389, 4443, 4505, 4506, 5000, 5060, 5061, 5222, 5269,
    5357, 5432, 5555, 5601, 5672, 5900, 5984, 6000, 6379, 6443, 6667, 7000,
    7077, 7474, 7687, 8000, 8008, 8080, 8081, 8086, 8088, 8090, 8123, 8161,
    8200, 8443, 8500, 8529, 8880, 8888, 8983, 9000, 9001, 9042, 9090, 9092,
    9100, 9200, 9300, 9418, 9999, 10000, 11211, 15672, 27017, 27018, 50000,
    50070, 61616,
}
PORT_RANDOM_MIN, PORT_RANDOM_MAX = 40000, 60000


# =============================================================
# I18N
# =============================================================
# Struktur: LOCALES[lang][key] = string
# Keys ohne Punkt-Konvention für Backend; im Frontend werden die gleichen
# Keys verwendet (mit t("key") Zugriff).
LOCALES: dict[str, dict[str, str]] = {
    "de": {
        # Backend / Setup
        "setup.title": "PC Overlay - Ersteinrichtung",
        "setup.config_path": "Konfig wird gespeichert in:",
        "setup.available_ips": "Verfügbare IPs:",
        "setup.recommended": "(empfohlen für LAN-Zugriff)",
        "setup.bind_ip": "Bind-IP",
        "setup.port": "Port",
        "setup.username": "Benutzer",
        "setup.password": "Passwort",
        "setup.password_repeat": "Passwort wiederholen",
        "setup.invalid_ip": "Ungültige IP '{ip}', nutze 0.0.0.0",
        "setup.invalid_port": "Port muss zwischen 1024 und 65535 liegen.",
        "setup.reserved_port": "Port {port} ist für bekannte Dienste reserviert.",
        "setup.port_busy": "Port {port} ist auf {host} belegt.",
        "setup.not_a_number": "Keine Zahl.",
        "setup.password_empty": "Passwort darf nicht leer sein.",
        "setup.password_mismatch": "Passwörter stimmen nicht überein.",
        "setup.done_title": "Einrichtung abgeschlossen",
        "setup.done_config": "Konfig:",
        "setup.done_bind": "Bind:",
        "setup.done_user": "Benutzer:",
        # Banner
        "banner.running": "{app} laeuft",
        "banner.bind": "Bind:",
        "banner.local": "Lokal:",
        "banner.network": "Netzwerk:",
        "banner.user": "Benutzer:",
        "banner.password": "Passwort: (nur als Hash in config.json)",
        "banner.ssh_targets": "SSH-Ziele gespeichert:",
        "banner.disk_display": "Disk-Anzeige: nur Medien >= {gb} GiB automatisch",
        "banner.lan_only": "Nur im lokalen Netzwerk erreichbar.",
        "banner.quit": "Beenden mit STRG+C.",
        "banner.stopped": "Beendet.",
        "banner.bind_error": "FEHLER: Konnte nicht an {ip}:{port} binden.",
        "banner.config_deleted": "Konfiguration gelöscht:",
        "banner.config_unreadable": "config.json konnte nicht gelesen werden:",
        # API-Fehler
        "err.auth": "Falscher Benutzer oder Passwort",
        "err.network": "Zugriff verweigert für {ip}. Nur LAN/VPN erlaubt.",
        "err.overlay_not_found": "Overlay '{name}' nicht gefunden",
        "err.no_ssh_target": "Kein SSH-Ziel namens '{name}'",
        "err.path_not_found": "Pfad nicht gefunden",
        "err.no_access": "Kein Zugriff",
        "err.no_file": "Keine Datei",
        "err.file_too_big": "Datei zu groß zum Bearbeiten im Browser",
        "err.path_missing": "Pfad fehlt",
        "err.no_write": "Keine Schreibrechte",
        "err.no_write_dir": "Keine Schreibrechte im Zielordner.",
        "err.save_failed": "Speichern fehlgeschlagen",
        "err.file_not_found": "Datei nicht gefunden",
        "err.no_paths": "Keine Pfade angegeben",
        "err.zip_failed": "ZIP-Erstellung fehlgeschlagen: {err}",
        "err.invalid_action": "Ungültige Aktion",
        "err.invalid_service": "Ungültiger Dienstname",
        "err.no_cmd": "Kein Befehl",
        "err.cmd_not_allowed": "Befehl nicht erlaubt",
        "err.delete_disabled": "Löschen ist deaktiviert",
        "err.name_required": "Pfad und neuer Name erforderlich",
        "err.invalid_name": "Ungültiger Name",
        "err.src_not_found": "Quelle nicht gefunden",
        "err.dst_exists": "Ziel existiert bereits",
        "err.mv_failed": "mv fehlgeschlagen",
        "err.mkdir_failed": "mkdir fehlgeschlagen",
        "err.mode_octal": "Modus muss oktal sein (z. B. 644)",
        "err.invalid_owner": "Ungültiger Besitzer (user[:group])",
        "err.chmod_failed": "chmod fehlgeschlagen",
        "err.chown_failed": "chown fehlgeschlagen",
        "err.local_reserved": "'local' ist reserviert",
        "err.fields_required": "Name, Host und Benutzer sind Pflichtfelder",
        "err.invalid_port": "Ungültiger Port",
        "err.auth_type": "auth muss 'password' oder 'key' sein",
        "err.not_found": "Nicht gefunden",
        "err.password_empty": "Passwort darf nicht leer sein",
        "err.invalid_bind": "Ungültige Bind-IP: {ip}",
        "err.port_not_allowed": "Port {port} ist nicht erlaubt.",
        "err.port_busy": "Port {port} ist belegt.",
        "err.no_valid_port": "Kein gültiger Port",
        "err.port_out_of_range": "Port außerhalb 1024–65535.",
        "err.port_reserved": "Port {port} ist reserviert.",
        "err.target_dir_invalid": "Zielordner existiert nicht oder ist kein Verzeichnis",
        "err.write_failed": "Schreiben fehlgeschlagen: {err}",
        "err.no_write_sudo": "Keine Schreibrechte für {path}. Sudo-Passwort setzen.",
        "err.tmp_write": "tmp write fehlgeschlagen: {err}",
        "err.mv_failed_sudo": "mv fehlgeschlagen: {err}",
        "err.sudo_upload": "Sudo-Upload fehlgeschlagen: {err}",
        "err.timeout": "Timeout nach {s}s",
        # Settings / Status
        "ok.conn": "Verbindung erfolgreich",
        "ok.conn_failed": "Verbindung fehlgeschlagen: {err}",
        "ok.sudo_works": "Sudo funktioniert",
        "ok.sudo_failed": "Sudo-Test fehlgeschlagen: {err}",
        "ok.port_free": "Port {port} ist frei und erlaubt.",
        "ok.saved_restart": "Gespeichert. Server neu starten, damit Änderungen wirksam werden.",
        "ok.test_failed": "Fehlgeschlagen: {err}",
        # Frontend - Header
        "ui.overlay": "Overlay",
        "ui.overlay_local": "auf diesem Rechner",
        "ui.tab_dash": "Dashboard",
        "ui.tab_proc": "Prozesse",
        "ui.tab_svc": "Dienste",
        "ui.tab_files": "Dateien",
        "ui.tab_shell": "Shell",
        "ui.tab_ssh": "SSH-Ziele",
        "ui.tab_settings": "Einstellungen",
        # Dashboard
        "ui.system": "System",
        "ui.hostname": "Hostname",
        "ui.os": "System",
        "ui.python": "Python",
        "ui.uptime": "Uptime",
        "ui.cpu_usage": "CPU-Auslastung",
        "ui.cores": "Kerne",
        "ui.ram": "RAM",
        "ui.bind_port": "Bind / Port",
        "ui.storage": "Speichermedien",
        "ui.edit_selection": "Auswahl bearbeiten",
        "ui.close_selection": "Auswahl schließen",
        "ui.disk_hint": "Standardmäßig werden nur Speichermedien ab 5 GiB angezeigt. Kleinere Medien erscheinen erst, wenn du sie hier manuell anhast.",
        "ui.save_selection": "Auswahl speichern",
        "ui.select_all": "Alle auswählen",
        "ui.select_none": "Keine auswählen",
        "ui.default_only_big": "Standard (nur ab 5 GiB)",
        "ui.cancel": "Abbrechen",
        "ui.no_disks": "Keine Speichermedien ausgewählt oder gefunden.",
        "ui.disk": "Disk",
        "ui.small": "klein",
        "ui.open_in_files": "In Dateien öffnen",
        "ui.auto_ge": "Auto ≥ {size}",
        "ui.manual": "Manuell",
        "ui.shown": "angezeigt",
        "ui.load_avg": "Load Average (1 / 5 / 15 min)",
        "ui.default_saved": "Standard gespeichert",
        "ui.selection_saved": "Auswahl gespeichert",
        "ui.default_restored": "Standard wiederhergestellt",
        # Prozesse
        "ui.filter_cmd": "Filter nach Name oder Befehl…",
        "ui.refresh": "Aktualisieren",
        "ui.pid": "PID",
        "ui.name": "Name",
        "ui.user": "User",
        "ui.cpu_pct": "CPU %",
        "ui.ram_pct": "RAM %",
        "ui.cmdline": "Cmdline",
        "ui.no_procs": "Keine Prozesse.",
        "ui.kill": "kill",
        "ui.kill_confirm_title": "PID {pid} beenden?",
        "ui.kill_confirm_msg": "SIGTERM wird gesendet.",
        "ui.kill_ok_label": "Beenden",
        "ui.killed": "PID {pid} beendet",
        # Dienste
        "ui.filter": "Filter…",
        "ui.load": "Load",
        "ui.active": "Active",
        "ui.sub": "Sub",
        "ui.no_services": "Keine Dienste.",
        "ui.start": "start",
        "ui.stop": "stop",
        "ui.restart": "restart",
        "ui.logs": "logs",
        "ui.logs_title": "Logs: {name}",
        "ui.close": "schließen",
        # Dateien
        "ui.path": "/pfad/zum/ordner",
        "ui.parent": "↑ Übergeordnet",
        "ui.open": "Öffnen",
        "ui.sudo_missing": "Dieses SSH-Ziel hat kein Sudo-Passwort. Schreibzugriffe in geschützte Ordner schlagen möglicherweise fehl.",
        "ui.sudo_ok": "Sudo aktiv – Schreibzugriffe werden bei Bedarf per sudo ausgeführt.",
        "ui.upload_title": "Datei(en) hochladen",
        "ui.upload_hint": "Klicken oder Datei hierher ziehen. Bei fehlenden Rechten wird sudo benutzt.",
        "ui.target_dir": "Aktueller Zielordner:",
        "ui.selected_n": "{n} ausgewählt",
        "ui.select_all_btn": "Alle auswählen",
        "ui.clear_selection": "Auswahl leeren",
        "ui.zip_selection": "Auswahl als ZIP",
        "ui.new_folder": "Neuer Ordner",
        "ui.delete_selection": "Auswahl löschen",
        "ui.perm": "Rechte",
        "ui.owner": "Besitzer",
        "ui.size": "Größe",
        "ui.modified": "Geändert",
        "ui.empty": "Leer.",
        "ui.edit": "bearbeiten",
        "ui.download": "Download",
        "ui.delete": "löschen",
        "ui.rename": "umbenennen",
        "ui.perms": "Rechte",
        "ui.chown": "Besitzer",
        "ui.save": "Speichern",
        "ui.saved": "Gespeichert",
        "ui.save_failed": "Speichern fehlgeschlagen",
        "ui.delete_confirm": "{n} Einträge löschen?",
        "ui.delete_recursive": "Ordner werden rekursiv entfernt.",
        "ui.delete_ok_label": "Löschen",
        "ui.deleted_n": "{n} gelöscht",
        "ui.deleted_failed": "{failed} von {total} fehlgeschlagen",
        "ui.deleted_one": "Gelöscht",
        "ui.delete_error": "Fehler beim Löschen",
        "ui.rename_title": "Umbenennen",
        "ui.rename_prompt": "Neuer Name für \"{name}\":",
        "ui.rename_ok_label": "Umbenennen",
        "ui.renamed": "Umbenannt",
        "ui.perms_title": "Rechte setzen für",
        "ui.perms_ok_label": "Setzen",
        "ui.invalid_mode": "Ungültiger Modus",
        "ui.perms_set": "Rechte gesetzt",
        "ui.chown_title": "Besitzer ändern",
        "ui.chown_prompt": "user[:group] für \"{path}\":",
        "ui.owner_set": "Besitzer gesetzt",
        "ui.new_folder_title": "Neuer Ordner in {dir}",
        "ui.new_folder_prompt": "Name:",
        "ui.create_label": "Anlegen",
        "ui.folder_created": "Ordner erstellt",
        "ui.zip_created": "ZIP erstellt",
        "ui.no_selection": "Keine Auswahl",
        "ui.upload_failed": "Fehler bei {name}: {err}",
        "ui.upload_processing": "Server verarbeitet…",
        "ui.upload_via_sudo": "via sudo",
        "ui.upload_total": "Gesamt",
        "ui.upload_error": "Fehler:",
        "ui.loading": "lade…",
        # Shell
        "ui.shell_placeholder": "Befehl eingeben… (z.B. uptime)",
        "ui.with_sudo": "mit sudo",
        "ui.run": "Ausführen",
        "ui.ready": "Bereit.",
        "ui.shell_running": "führe auf {target} aus:",
        # SSH
        "ui.ssh_warning": "SSH-Passwörter, Sudo-Passwörter und Key-Passphrasen werden in config.json im Klartext gespeichert. Schütze die Datei entsprechend (chmod 600).",
        "ui.add_edit_ssh": "SSH-Ziel hinzufügen / bearbeiten",
        "ui.host_ip": "Host / IP",
        "ui.auth_method": "Auth-Methode",
        "ui.auth_password": "Passwort",
        "ui.auth_key": "SSH-Key",
        "ui.ssh_password": "SSH-Passwort (wird gespeichert)",
        "ui.ssh_key_path": "Pfad zum privaten Key",
        "ui.ssh_key_pass": "Key-Passphrase (optional)",
        "ui.ssh_key_pass_ph": "leer, wenn unverschlüsselt",
        "ui.sudo_password": "Sudo-Passwort (optional – für Schreibzugriffe in /etc, /var, …)",
        "ui.sudo_password_ph": "leer lassen, wenn kein sudo nötig",
        "ui.clear_sudo": "Sudo-Passwort löschen",
        "ui.save_test": "Speichern & testen",
        "ui.clear_form": "Formular leeren",
        "ui.saved_ssh": "Gespeichert – {msg}",
        "ui.stored_targets": "Gespeicherte SSH-Ziele",
        "ui.port": "Port",
        "ui.auth": "Auth",
        "ui.sudo": "Sudo",
        "ui.yes": "aktiv",
        "ui.no": "nein",
        "ui.test": "testen",
        "ui.edit_btn": "bearbeiten",
        "ui.sudo_password_btn": "Sudo-Passwort",
        "ui.delete_btn": "löschen",
        "ui.no_targets": "Keine SSH-Ziele gespeichert.",
        "ui.target_deleted": "Gelöscht",
        "ui.target_delete_title": "SSH-Ziel löschen?",
        "ui.form_loaded": "Formular geladen.",
        "ui.fields_required": "Name, Host und Benutzer sind Pflicht",
        "ui.sudo_pw_title": "Sudo-Passwort",
        "ui.sudo_pw_prompt": "Passwort für \"{name}\" (wird gespeichert):",
        "ui.set": "Setzen",
        # Settings
        "ui.current_config": "Aktuelle Konfiguration",
        "ui.bind_addr_port": "Bind-Adresse & Port",
        "ui.bind_ip": "Bind-IP",
        "ui.custom_ip": "Oder eigene IP eingeben",
        "ui.custom_ip_ph": "z.B. 192.168.1.100",
        "ui.random_port": "Zufälliger Port",
        "ui.check_port": "Port prüfen",
        "ui.change_login": "Login ändern (optional)",
        "ui.new_user_ph": "Neuer Benutzername",
        "ui.new_pass_ph": "Neues Passwort",
        "ui.new_pass2_ph": "Passwort wiederholen",
        "ui.allow_delete": "Löschen im Datei-Browser erlauben",
        "ui.reset": "Zurücksetzen",
        "ui.restart_note": "Nach dem Speichern: Server mit STRG+C beenden und erneut starten.",
        "ui.port_random": "Zufälliger Port gewählt.",
        "ui.port_enter": "Bitte Port eingeben.",
        "ui.checking": "prüfe…",
        "ui.password_mismatch": "Passwörter stimmen nicht überein",
        "ui.no_port": "Kein Port angegeben",
        "ui.config_saved": "Gespeichert.\nNach Neustart:\n{url}",
        # Generisch
        "ui.error": "Fehler: {err}",
        "ui.language": "Sprache",
    },
    "en": {
        "setup.title": "PC Overlay - Initial setup",
        "setup.config_path": "Config will be saved to:",
        "setup.available_ips": "Available IPs:",
        "setup.recommended": "(recommended for LAN access)",
        "setup.bind_ip": "Bind IP",
        "setup.port": "Port",
        "setup.username": "Username",
        "setup.password": "Password",
        "setup.password_repeat": "Repeat password",
        "setup.invalid_ip": "Invalid IP '{ip}', using 0.0.0.0",
        "setup.invalid_port": "Port must be between 1024 and 65535.",
        "setup.reserved_port": "Port {port} is reserved for well-known services.",
        "setup.port_busy": "Port {port} is already in use on {host}.",
        "setup.not_a_number": "Not a number.",
        "setup.password_empty": "Password must not be empty.",
        "setup.password_mismatch": "Passwords do not match.",
        "setup.done_title": "Setup complete",
        "setup.done_config": "Config:",
        "setup.done_bind": "Bind:",
        "setup.done_user": "User:",
        "banner.running": "{app} running",
        "banner.bind": "Bind:",
        "banner.local": "Local:",
        "banner.network": "Network:",
        "banner.user": "User:",
        "banner.password": "Password: (hash only, stored in config.json)",
        "banner.ssh_targets": "Stored SSH targets:",
        "banner.disk_display": "Disk display: only media >= {gb} GiB automatically",
        "banner.lan_only": "Reachable on the local network only.",
        "banner.quit": "Press CTRL+C to quit.",
        "banner.stopped": "Stopped.",
        "banner.bind_error": "ERROR: Could not bind to {ip}:{port}.",
        "banner.config_deleted": "Configuration deleted:",
        "banner.config_unreadable": "Could not read config.json:",
        "err.auth": "Invalid username or password",
        "err.network": "Access denied for {ip}. LAN/VPN only.",
        "err.overlay_not_found": "Overlay '{name}' not found",
        "err.no_ssh_target": "No SSH target named '{name}'",
        "err.path_not_found": "Path not found",
        "err.no_access": "No access",
        "err.no_file": "Not a file",
        "err.file_too_big": "File too large to edit in the browser",
        "err.path_missing": "Path missing",
        "err.no_write": "No write permission",
        "err.no_write_dir": "No write permission in target folder.",
        "err.save_failed": "Save failed",
        "err.file_not_found": "File not found",
        "err.no_paths": "No paths provided",
        "err.zip_failed": "ZIP creation failed: {err}",
        "err.invalid_action": "Invalid action",
        "err.invalid_service": "Invalid service name",
        "err.no_cmd": "No command",
        "err.cmd_not_allowed": "Command not allowed",
        "err.delete_disabled": "Deletion is disabled",
        "err.name_required": "Path and new name are required",
        "err.invalid_name": "Invalid name",
        "err.src_not_found": "Source not found",
        "err.dst_exists": "Target already exists",
        "err.mv_failed": "mv failed",
        "err.mkdir_failed": "mkdir failed",
        "err.mode_octal": "Mode must be octal (e.g. 644)",
        "err.invalid_owner": "Invalid owner (user[:group])",
        "err.chmod_failed": "chmod failed",
        "err.chown_failed": "chown failed",
        "err.local_reserved": "'local' is reserved",
        "err.fields_required": "Name, host and user are required",
        "err.invalid_port": "Invalid port",
        "err.auth_type": "auth must be 'password' or 'key'",
        "err.not_found": "Not found",
        "err.password_empty": "Password must not be empty",
        "err.invalid_bind": "Invalid bind IP: {ip}",
        "err.port_not_allowed": "Port {port} is not allowed.",
        "err.port_busy": "Port {port} is already in use.",
        "err.no_valid_port": "Not a valid port",
        "err.port_out_of_range": "Port outside 1024–65535.",
        "err.port_reserved": "Port {port} is reserved.",
        "err.target_dir_invalid": "Target folder does not exist or is not a directory",
        "err.write_failed": "Write failed: {err}",
        "err.no_write_sudo": "No write permission for {path}. Set a sudo password.",
        "err.tmp_write": "tmp write failed: {err}",
        "err.mv_failed_sudo": "mv failed: {err}",
        "err.sudo_upload": "Sudo upload failed: {err}",
        "err.timeout": "Timeout after {s}s",
        "ok.conn": "Connection successful",
        "ok.conn_failed": "Connection failed: {err}",
        "ok.sudo_works": "Sudo works",
        "ok.sudo_failed": "Sudo test failed: {err}",
        "ok.port_free": "Port {port} is free and allowed.",
        "ok.saved_restart": "Saved. Restart the server for changes to take effect.",
        "ok.test_failed": "Failed: {err}",
        "ui.overlay": "Overlay",
        "ui.overlay_local": "on this machine",
        "ui.tab_dash": "Dashboard",
        "ui.tab_proc": "Processes",
        "ui.tab_svc": "Services",
        "ui.tab_files": "Files",
        "ui.tab_shell": "Shell",
        "ui.tab_ssh": "SSH targets",
        "ui.tab_settings": "Settings",
        "ui.system": "System",
        "ui.hostname": "Hostname",
        "ui.os": "OS",
        "ui.python": "Python",
        "ui.uptime": "Uptime",
        "ui.cpu_usage": "CPU usage",
        "ui.cores": "cores",
        "ui.ram": "RAM",
        "ui.bind_port": "Bind / port",
        "ui.storage": "Storage",
        "ui.edit_selection": "Edit selection",
        "ui.close_selection": "Close selection",
        "ui.disk_hint": "By default, only media of 5 GiB or larger is shown. Smaller media appear once you check them here.",
        "ui.save_selection": "Save selection",
        "ui.select_all": "Select all",
        "ui.select_none": "Select none",
        "ui.default_only_big": "Default (only ≥ 5 GiB)",
        "ui.cancel": "Cancel",
        "ui.no_disks": "No storage media selected or found.",
        "ui.disk": "Disk",
        "ui.small": "small",
        "ui.open_in_files": "Open in files",
        "ui.auto_ge": "Auto ≥ {size}",
        "ui.manual": "Manual",
        "ui.shown": "shown",
        "ui.load_avg": "Load average (1 / 5 / 15 min)",
        "ui.default_saved": "Default saved",
        "ui.selection_saved": "Selection saved",
        "ui.default_restored": "Default restored",
        "ui.filter_cmd": "Filter by name or command…",
        "ui.refresh": "Refresh",
        "ui.pid": "PID",
        "ui.name": "Name",
        "ui.user": "User",
        "ui.cpu_pct": "CPU %",
        "ui.ram_pct": "RAM %",
        "ui.cmdline": "Cmdline",
        "ui.no_procs": "No processes.",
        "ui.kill": "kill",
        "ui.kill_confirm_title": "Terminate PID {pid}?",
        "ui.kill_confirm_msg": "SIGTERM will be sent.",
        "ui.kill_ok_label": "Terminate",
        "ui.killed": "PID {pid} terminated",
        "ui.filter": "Filter…",
        "ui.load": "Load",
        "ui.active": "Active",
        "ui.sub": "Sub",
        "ui.no_services": "No services.",
        "ui.start": "start",
        "ui.stop": "stop",
        "ui.restart": "restart",
        "ui.logs": "logs",
        "ui.logs_title": "Logs: {name}",
        "ui.close": "close",
        "ui.path": "/path/to/folder",
        "ui.parent": "↑ Parent",
        "ui.open": "Open",
        "ui.sudo_missing": "This SSH target has no sudo password. Writes to protected folders may fail.",
        "ui.sudo_ok": "Sudo enabled – writes will be performed via sudo when needed.",
        "ui.upload_title": "Upload file(s)",
        "ui.upload_hint": "Click or drag a file here. Sudo is used when permissions are missing.",
        "ui.target_dir": "Current target folder:",
        "ui.selected_n": "{n} selected",
        "ui.select_all_btn": "Select all",
        "ui.clear_selection": "Clear selection",
        "ui.zip_selection": "Selection as ZIP",
        "ui.new_folder": "New folder",
        "ui.delete_selection": "Delete selection",
        "ui.perm": "Mode",
        "ui.owner": "Owner",
        "ui.size": "Size",
        "ui.modified": "Modified",
        "ui.empty": "Empty.",
        "ui.edit": "edit",
        "ui.download": "Download",
        "ui.delete": "delete",
        "ui.rename": "rename",
        "ui.perms": "Mode",
        "ui.chown": "Owner",
        "ui.save": "Save",
        "ui.saved": "Saved",
        "ui.save_failed": "Save failed",
        "ui.delete_confirm": "Delete {n} entries?",
        "ui.delete_recursive": "Folders will be removed recursively.",
        "ui.delete_ok_label": "Delete",
        "ui.deleted_n": "{n} deleted",
        "ui.deleted_failed": "{failed} of {total} failed",
        "ui.deleted_one": "Deleted",
        "ui.delete_error": "Delete failed",
        "ui.rename_title": "Rename",
        "ui.rename_prompt": "New name for \"{name}\":",
        "ui.rename_ok_label": "Rename",
        "ui.renamed": "Renamed",
        "ui.perms_title": "Set permissions for",
        "ui.perms_ok_label": "Set",
        "ui.invalid_mode": "Invalid mode",
        "ui.perms_set": "Permissions set",
        "ui.chown_title": "Change owner",
        "ui.chown_prompt": "user[:group] for \"{path}\":",
        "ui.owner_set": "Owner set",
        "ui.new_folder_title": "New folder in {dir}",
        "ui.new_folder_prompt": "Name:",
        "ui.create_label": "Create",
        "ui.folder_created": "Folder created",
        "ui.zip_created": "ZIP created",
        "ui.no_selection": "Nothing selected",
        "ui.upload_failed": "Error for {name}: {err}",
        "ui.upload_processing": "Server processing…",
        "ui.upload_via_sudo": "via sudo",
        "ui.upload_total": "Total",
        "ui.upload_error": "Error:",
        "ui.loading": "loading…",
        "ui.shell_placeholder": "Enter command… (e.g. uptime)",
        "ui.with_sudo": "with sudo",
        "ui.run": "Run",
        "ui.ready": "Ready.",
        "ui.shell_running": "running on {target}:",
        "ui.ssh_warning": "SSH passwords, sudo passwords and key passphrases are stored in config.json in plain text. Protect the file accordingly (chmod 600).",
        "ui.add_edit_ssh": "Add / edit SSH target",
        "ui.host_ip": "Host / IP",
        "ui.auth_method": "Auth method",
        "ui.auth_password": "Password",
        "ui.auth_key": "SSH key",
        "ui.ssh_password": "SSH password (will be stored)",
        "ui.ssh_key_path": "Path to private key",
        "ui.ssh_key_pass": "Key passphrase (optional)",
        "ui.ssh_key_pass_ph": "empty if unencrypted",
        "ui.sudo_password": "Sudo password (optional – for writes to /etc, /var, …)",
        "ui.sudo_password_ph": "leave empty if sudo is not needed",
        "ui.clear_sudo": "Remove sudo password",
        "ui.save_test": "Save & test",
        "ui.clear_form": "Clear form",
        "ui.saved_ssh": "Saved – {msg}",
        "ui.stored_targets": "Stored SSH targets",
        "ui.port": "Port",
        "ui.auth": "Auth",
        "ui.sudo": "Sudo",
        "ui.yes": "active",
        "ui.no": "no",
        "ui.test": "test",
        "ui.edit_btn": "edit",
        "ui.sudo_password_btn": "Sudo password",
        "ui.delete_btn": "delete",
        "ui.no_targets": "No SSH targets stored.",
        "ui.target_deleted": "Deleted",
        "ui.target_delete_title": "Delete SSH target?",
        "ui.form_loaded": "Form loaded.",
        "ui.fields_required": "Name, host and user are required",
        "ui.sudo_pw_title": "Sudo password",
        "ui.sudo_pw_prompt": "Password for \"{name}\" (will be stored):",
        "ui.set": "Set",
        "ui.current_config": "Current configuration",
        "ui.bind_addr_port": "Bind address & port",
        "ui.bind_ip": "Bind IP",
        "ui.custom_ip": "Or enter a custom IP",
        "ui.custom_ip_ph": "e.g. 192.168.1.100",
        "ui.random_port": "Random port",
        "ui.check_port": "Check port",
        "ui.change_login": "Change login (optional)",
        "ui.new_user_ph": "New username",
        "ui.new_pass_ph": "New password",
        "ui.new_pass2_ph": "Repeat password",
        "ui.allow_delete": "Allow deletion in file browser",
        "ui.reset": "Reset",
        "ui.restart_note": "After saving: stop the server with CTRL+C and start it again.",
        "ui.port_random": "Random port chosen.",
        "ui.port_enter": "Please enter a port.",
        "ui.checking": "checking…",
        "ui.password_mismatch": "Passwords do not match",
        "ui.no_port": "No port specified",
        "ui.config_saved": "Saved.\nAfter restart:\n{url}",
        "ui.error": "Error: {err}",
        "ui.language": "Language",
    },
    "ru": {
        "setup.title": "PC Overlay - Первичная настройка",
        "setup.config_path": "Конфигурация будет сохранена в:",
        "setup.available_ips": "Доступные IP-адреса:",
        "setup.recommended": "(рекомендуется для доступа из локальной сети)",
        "setup.bind_ip": "IP-адрес привязки",
        "setup.port": "Порт",
        "setup.username": "Имя пользователя",
        "setup.password": "Пароль",
        "setup.password_repeat": "Повторите пароль",
        "setup.invalid_ip": "Недопустимый IP '{ip}', используется 0.0.0.0",
        "setup.invalid_port": "Порт должен быть в диапазоне от 1024 до 65535.",
        "setup.reserved_port": "Порт {port} зарезервирован за известными службами.",
        "setup.port_busy": "Порт {port} уже занят на {host}.",
        "setup.not_a_number": "Не число.",
        "setup.password_empty": "Пароль не может быть пустым.",
        "setup.password_mismatch": "Пароли не совпадают.",
        "setup.done_title": "Настройка завершена",
        "setup.done_config": "Конфигурация:",
        "setup.done_bind": "Привязка:",
        "setup.done_user": "Пользователь:",
        "banner.running": "{app} запущен",
        "banner.bind": "Привязка:",
        "banner.local": "Локально:",
        "banner.network": "Сеть:",
        "banner.user": "Пользователь:",
        "banner.password": "Пароль: (только хэш, хранится в config.json)",
        "banner.ssh_targets": "Сохранённых SSH-целей:",
        "banner.disk_display": "Отображение дисков: автоматически только носители ≥ {gb} ГиБ",
        "banner.lan_only": "Доступно только из локальной сети.",
        "banner.quit": "Нажмите CTRL+C для выхода.",
        "banner.stopped": "Остановлено.",
        "banner.bind_error": "ОШИБКА: Не удалось привязаться к {ip}:{port}.",
        "banner.config_deleted": "Конфигурация удалена:",
        "banner.config_unreadable": "Не удалось прочитать config.json:",
        "err.auth": "Неверное имя пользователя или пароль",
        "err.network": "Доступ запрещён для {ip}. Только LAN/VPN.",
        "err.overlay_not_found": "Оверлей '{name}' не найден",
        "err.no_ssh_target": "Нет SSH-цели с именем '{name}'",
        "err.path_not_found": "Путь не найден",
        "err.no_access": "Нет доступа",
        "err.no_file": "Не файл",
        "err.file_too_big": "Файл слишком большой для редактирования в браузере",
        "err.path_missing": "Путь отсутствует",
        "err.no_write": "Нет прав на запись",
        "err.no_write_dir": "Нет прав на запись в целевую папку.",
        "err.save_failed": "Сохранение не удалось",
        "err.file_not_found": "Файл не найден",
        "err.no_paths": "Пути не указаны",
        "err.zip_failed": "Не удалось создать ZIP: {err}",
        "err.invalid_action": "Недопустимое действие",
        "err.invalid_service": "Недопустимое имя службы",
        "err.no_cmd": "Нет команды",
        "err.cmd_not_allowed": "Команда не разрешена",
        "err.delete_disabled": "Удаление отключено",
        "err.name_required": "Требуется путь и новое имя",
        "err.invalid_name": "Недопустимое имя",
        "err.src_not_found": "Источник не найден",
        "err.dst_exists": "Цель уже существует",
        "err.mv_failed": "mv не удалось",
        "err.mkdir_failed": "mkdir не удалось",
        "err.mode_octal": "Режим должен быть восьмеричным (например, 644)",
        "err.invalid_owner": "Недопустимый владелец (user[:group])",
        "err.chmod_failed": "chmod не удалось",
        "err.chown_failed": "chown не удалось",
        "err.local_reserved": "'local' зарезервировано",
        "err.fields_required": "Требуются имя, хост и пользователь",
        "err.invalid_port": "Недопустимый порт",
        "err.auth_type": "auth должен быть 'password' или 'key'",
        "err.not_found": "Не найдено",
        "err.password_empty": "Пароль не может быть пустым",
        "err.invalid_bind": "Недопустимый IP привязки: {ip}",
        "err.port_not_allowed": "Порт {port} не разрешён.",
        "err.port_busy": "Порт {port} уже занят.",
        "err.no_valid_port": "Недопустимый порт",
        "err.port_out_of_range": "Порт вне диапазона 1024–65535.",
        "err.port_reserved": "Порт {port} зарезервирован.",
        "err.target_dir_invalid": "Целевая папка не существует или не является папкой",
        "err.write_failed": "Ошибка записи: {err}",
        "err.no_write_sudo": "Нет прав на запись для {path}. Укажите пароль sudo.",
        "err.tmp_write": "ошибка записи tmp: {err}",
        "err.mv_failed_sudo": "mv не удалось: {err}",
        "err.sudo_upload": "Загрузка через sudo не удалась: {err}",
        "err.timeout": "Тайм-аут после {s} с",
        "ok.conn": "Соединение установлено",
        "ok.conn_failed": "Соединение не удалось: {err}",
        "ok.sudo_works": "Sudo работает",
        "ok.sudo_failed": "Проверка sudo не удалась: {err}",
        "ok.port_free": "Порт {port} свободен и разрешён.",
        "ok.saved_restart": "Сохранено. Перезапустите сервер, чтобы применить изменения.",
        "ok.test_failed": "Не удалось: {err}",
        "ui.overlay": "Оверлей",
        "ui.overlay_local": "на этом компьютере",
        "ui.tab_dash": "Панель",
        "ui.tab_proc": "Процессы",
        "ui.tab_svc": "Службы",
        "ui.tab_files": "Файлы",
        "ui.tab_shell": "Оболочка",
        "ui.tab_ssh": "SSH-цели",
        "ui.tab_settings": "Настройки",
        "ui.system": "Система",
        "ui.hostname": "Имя хоста",
        "ui.os": "ОС",
        "ui.python": "Python",
        "ui.uptime": "Время работы",
        "ui.cpu_usage": "Загрузка CPU",
        "ui.cores": "ядер",
        "ui.ram": "ОЗУ",
        "ui.bind_port": "Привязка / порт",
        "ui.storage": "Накопители",
        "ui.edit_selection": "Изменить выбор",
        "ui.close_selection": "Закрыть выбор",
        "ui.disk_hint": "По умолчанию отображаются только носители от 5 ГиБ. Меньшие появятся, если отметить их здесь.",
        "ui.save_selection": "Сохранить выбор",
        "ui.select_all": "Выбрать все",
        "ui.select_none": "Снять выбор",
        "ui.default_only_big": "По умолчанию (только ≥ 5 ГиБ)",
        "ui.cancel": "Отмена",
        "ui.no_disks": "Носители не выбраны или не найдены.",
        "ui.disk": "Диск",
        "ui.small": "малый",
        "ui.open_in_files": "Открыть в файлах",
        "ui.auto_ge": "Авто ≥ {size}",
        "ui.manual": "Вручную",
        "ui.shown": "показано",
        "ui.load_avg": "Средняя нагрузка (1 / 5 / 15 мин)",
        "ui.default_saved": "По умолчанию сохранено",
        "ui.selection_saved": "Выбор сохранён",
        "ui.default_restored": "По умолчанию восстановлено",
        "ui.filter_cmd": "Фильтр по имени или команде…",
        "ui.refresh": "Обновить",
        "ui.pid": "PID",
        "ui.name": "Имя",
        "ui.user": "Пользователь",
        "ui.cpu_pct": "CPU %",
        "ui.ram_pct": "RAM %",
        "ui.cmdline": "Командная строка",
        "ui.no_procs": "Нет процессов.",
        "ui.kill": "завершить",
        "ui.kill_confirm_title": "Завершить PID {pid}?",
        "ui.kill_confirm_msg": "Будет отправлен SIGTERM.",
        "ui.kill_ok_label": "Завершить",
        "ui.killed": "PID {pid} завершён",
        "ui.filter": "Фильтр…",
        "ui.load": "Загрузка",
        "ui.active": "Активно",
        "ui.sub": "Подсостояние",
        "ui.no_services": "Нет служб.",
        "ui.start": "запуск",
        "ui.stop": "стоп",
        "ui.restart": "перезапуск",
        "ui.logs": "логи",
        "ui.logs_title": "Логи: {name}",
        "ui.close": "закрыть",
        "ui.path": "/путь/к/папке",
        "ui.parent": "↑ Родительская",
        "ui.open": "Открыть",
        "ui.sudo_missing": "У этой SSH-цели нет пароля sudo. Запись в защищённые папки может не работать.",
        "ui.sudo_ok": "Sudo активен — запись при необходимости выполняется через sudo.",
        "ui.upload_title": "Загрузить файл(ы)",
        "ui.upload_hint": "Нажмите или перетащите файл сюда. При отсутствии прав используется sudo.",
        "ui.target_dir": "Текущая целевая папка:",
        "ui.selected_n": "Выбрано: {n}",
        "ui.select_all_btn": "Выбрать все",
        "ui.clear_selection": "Очистить выбор",
        "ui.zip_selection": "Выбор в ZIP",
        "ui.new_folder": "Новая папка",
        "ui.delete_selection": "Удалить выбранное",
        "ui.perm": "Права",
        "ui.owner": "Владелец",
        "ui.size": "Размер",
        "ui.modified": "Изменён",
        "ui.empty": "Пусто.",
        "ui.edit": "изменить",
        "ui.download": "Скачать",
        "ui.delete": "удалить",
        "ui.rename": "переименовать",
        "ui.perms": "Права",
        "ui.chown": "Владелец",
        "ui.save": "Сохранить",
        "ui.saved": "Сохранено",
        "ui.save_failed": "Сохранение не удалось",
        "ui.delete_confirm": "Удалить {n} записей?",
        "ui.delete_recursive": "Папки будут удалены рекурсивно.",
        "ui.delete_ok_label": "Удалить",
        "ui.deleted_n": "Удалено: {n}",
        "ui.deleted_failed": "Не удалось: {failed} из {total}",
        "ui.deleted_one": "Удалено",
        "ui.delete_error": "Ошибка удаления",
        "ui.rename_title": "Переименовать",
        "ui.rename_prompt": "Новое имя для \"{name}\":",
        "ui.rename_ok_label": "Переименовать",
        "ui.renamed": "Переименовано",
        "ui.perms_title": "Установить права для",
        "ui.perms_ok_label": "Установить",
        "ui.invalid_mode": "Недопустимый режим",
        "ui.perms_set": "Права установлены",
        "ui.chown_title": "Сменить владельца",
        "ui.chown_prompt": "user[:group] для \"{path}\":",
        "ui.owner_set": "Владелец установлен",
        "ui.new_folder_title": "Новая папка в {dir}",
        "ui.new_folder_prompt": "Имя:",
        "ui.create_label": "Создать",
        "ui.folder_created": "Папка создана",
        "ui.zip_created": "ZIP создан",
        "ui.no_selection": "Ничего не выбрано",
        "ui.upload_failed": "Ошибка для {name}: {err}",
        "ui.upload_processing": "Обработка на сервере…",
        "ui.upload_via_sudo": "через sudo",
        "ui.upload_total": "Всего",
        "ui.upload_error": "Ошибка:",
        "ui.loading": "загрузка…",
        "ui.shell_placeholder": "Введите команду… (например, uptime)",
        "ui.with_sudo": "с sudo",
        "ui.run": "Выполнить",
        "ui.ready": "Готов.",
        "ui.shell_running": "выполняется на {target}:",
        "ui.ssh_warning": "Пароли SSH, sudo и парольные фразы ключей хранятся в config.json в открытом виде. Защитите файл (chmod 600).",
        "ui.add_edit_ssh": "Добавить / изменить SSH-цель",
        "ui.host_ip": "Хост / IP",
        "ui.auth_method": "Метод аутентификации",
        "ui.auth_password": "Пароль",
        "ui.auth_key": "SSH-ключ",
        "ui.ssh_password": "Пароль SSH (будет сохранён)",
        "ui.ssh_key_path": "Путь к приватному ключу",
        "ui.ssh_key_pass": "Парольная фраза ключа (необязательно)",
        "ui.ssh_key_pass_ph": "пусто, если не зашифрован",
        "ui.sudo_password": "Пароль sudo (необязательно — для записи в /etc, /var, …)",
        "ui.sudo_password_ph": "оставьте пустым, если sudo не нужен",
        "ui.clear_sudo": "Удалить пароль sudo",
        "ui.save_test": "Сохранить и проверить",
        "ui.clear_form": "Очистить форму",
        "ui.saved_ssh": "Сохранено – {msg}",
        "ui.stored_targets": "Сохранённые SSH-цели",
        "ui.port": "Порт",
        "ui.auth": "Аутентификация",
        "ui.sudo": "Sudo",
        "ui.yes": "активно",
        "ui.no": "нет",
        "ui.test": "проверить",
        "ui.edit_btn": "изменить",
        "ui.sudo_password_btn": "Пароль sudo",
        "ui.delete_btn": "удалить",
        "ui.no_targets": "SSH-цели не сохранены.",
        "ui.target_deleted": "Удалено",
        "ui.target_delete_title": "Удалить SSH-цель?",
        "ui.form_loaded": "Форма загружена.",
        "ui.fields_required": "Требуются имя, хост и пользователь",
        "ui.sudo_pw_title": "Пароль sudo",
        "ui.sudo_pw_prompt": "Пароль для \"{name}\" (будет сохранён):",
        "ui.set": "Установить",
        "ui.current_config": "Текущая конфигурация",
        "ui.bind_addr_port": "Адрес привязки и порт",
        "ui.bind_ip": "IP привязки",
        "ui.custom_ip": "Или введите собственный IP",
        "ui.custom_ip_ph": "например, 192.168.1.100",
        "ui.random_port": "Случайный порт",
        "ui.check_port": "Проверить порт",
        "ui.change_login": "Изменить логин (необязательно)",
        "ui.new_user_ph": "Новое имя пользователя",
        "ui.new_pass_ph": "Новый пароль",
        "ui.new_pass2_ph": "Повторите пароль",
        "ui.allow_delete": "Разрешить удаление в файловом браузере",
        "ui.reset": "Сбросить",
        "ui.restart_note": "После сохранения: остановите сервер (CTRL+C) и запустите снова.",
        "ui.port_random": "Выбран случайный порт.",
        "ui.port_enter": "Введите порт.",
        "ui.checking": "проверка…",
        "ui.password_mismatch": "Пароли не совпадают",
        "ui.no_port": "Порт не указан",
        "ui.config_saved": "Сохранено.\nПосле перезапуска:\n{url}",
        "ui.error": "Ошибка: {err}",
        "ui.language": "Язык",
    },
    "ja": {
        "setup.title": "PC Overlay - 初期設定",
        "setup.config_path": "設定の保存先:",
        "setup.available_ips": "利用可能な IP:",
        "setup.recommended": "(LAN アクセスに推奨)",
        "setup.bind_ip": "バインド IP",
        "setup.port": "ポート",
        "setup.username": "ユーザー名",
        "setup.password": "パスワード",
        "setup.password_repeat": "パスワードを再入力",
        "setup.invalid_ip": "無効な IP '{ip}'、0.0.0.0 を使用します",
        "setup.invalid_port": "ポートは 1024〜65535 の範囲で指定してください。",
        "setup.reserved_port": "ポート {port} は既知のサービス用に予約されています。",
        "setup.port_busy": "ポート {port} は {host} で使用中です。",
        "setup.not_a_number": "数値ではありません。",
        "setup.password_empty": "パスワードを空にできません。",
        "setup.password_mismatch": "パスワードが一致しません。",
        "setup.done_title": "セットアップ完了",
        "setup.done_config": "設定:",
        "setup.done_bind": "バインド:",
        "setup.done_user": "ユーザー:",
        "banner.running": "{app} 起動中",
        "banner.bind": "バインド:",
        "banner.local": "ローカル:",
        "banner.network": "ネットワーク:",
        "banner.user": "ユーザー:",
        "banner.password": "パスワード: (ハッシュのみ config.json に保存)",
        "banner.ssh_targets": "保存された SSH ターゲット:",
        "banner.disk_display": "ディスク表示: {gb} GiB 以上のみ自動表示",
        "banner.lan_only": "ローカルネットワークからのみアクセス可能。",
        "banner.quit": "CTRL+C で終了。",
        "banner.stopped": "停止しました。",
        "banner.bind_error": "エラー: {ip}:{port} にバインドできませんでした。",
        "banner.config_deleted": "設定を削除しました:",
        "banner.config_unreadable": "config.json を読み込めませんでした:",
        "err.auth": "ユーザー名またはパスワードが正しくありません",
        "err.network": "{ip} からのアクセスを拒否しました。LAN/VPN のみ。",
        "err.overlay_not_found": "オーバーレイ '{name}' が見つかりません",
        "err.no_ssh_target": "'{name}' という SSH ターゲットがありません",
        "err.path_not_found": "パスが見つかりません",
        "err.no_access": "アクセス不可",
        "err.no_file": "ファイルではありません",
        "err.file_too_big": "ブラウザで編集するにはファイルが大きすぎます",
        "err.path_missing": "パスがありません",
        "err.no_write": "書き込み権限がありません",
        "err.no_write_dir": "対象フォルダに書き込み権限がありません。",
        "err.save_failed": "保存に失敗しました",
        "err.file_not_found": "ファイルが見つかりません",
        "err.no_paths": "パスが指定されていません",
        "err.zip_failed": "ZIP 作成に失敗: {err}",
        "err.invalid_action": "無効な操作",
        "err.invalid_service": "無効なサービス名",
        "err.no_cmd": "コマンドがありません",
        "err.cmd_not_allowed": "コマンドは許可されていません",
        "err.delete_disabled": "削除は無効です",
        "err.name_required": "パスと新しい名前が必要です",
        "err.invalid_name": "無効な名前",
        "err.src_not_found": "コピー元が見つかりません",
        "err.dst_exists": "コピー先が既に存在します",
        "err.mv_failed": "mv に失敗しました",
        "err.mkdir_failed": "mkdir に失敗しました",
        "err.mode_octal": "モードは 8 進数で指定してください (例: 644)",
        "err.invalid_owner": "無効な所有者 (user[:group])",
        "err.chmod_failed": "chmod に失敗しました",
        "err.chown_failed": "chown に失敗しました",
        "err.local_reserved": "'local' は予約済みです",
        "err.fields_required": "名前・ホスト・ユーザーは必須です",
        "err.invalid_port": "無効なポート",
        "err.auth_type": "auth は 'password' または 'key' である必要があります",
        "err.not_found": "見つかりません",
        "err.password_empty": "パスワードを空にできません",
        "err.invalid_bind": "無効なバインド IP: {ip}",
        "err.port_not_allowed": "ポート {port} は許可されていません。",
        "err.port_busy": "ポート {port} は使用中です。",
        "err.no_valid_port": "有効なポートではありません",
        "err.port_out_of_range": "ポートが 1024〜65535 の範囲外です。",
        "err.port_reserved": "ポート {port} は予約されています。",
        "err.target_dir_invalid": "対象フォルダが存在しないか、ディレクトリではありません",
        "err.write_failed": "書き込みに失敗: {err}",
        "err.no_write_sudo": "{path} への書き込み権限がありません。sudo パスワードを設定してください。",
        "err.tmp_write": "tmp 書き込みに失敗: {err}",
        "err.mv_failed_sudo": "mv に失敗: {err}",
        "err.sudo_upload": "sudo アップロードに失敗: {err}",
        "err.timeout": "{s} 秒でタイムアウト",
        "ok.conn": "接続に成功しました",
        "ok.conn_failed": "接続に失敗: {err}",
        "ok.sudo_works": "sudo は機能しています",
        "ok.sudo_failed": "sudo テストに失敗: {err}",
        "ok.port_free": "ポート {port} は空いており、許可されています。",
        "ok.saved_restart": "保存しました。変更を反映するにはサーバーを再起動してください。",
        "ok.test_failed": "失敗: {err}",
        "ui.overlay": "オーバーレイ",
        "ui.overlay_local": "このマシン上",
        "ui.tab_dash": "ダッシュボード",
        "ui.tab_proc": "プロセス",
        "ui.tab_svc": "サービス",
        "ui.tab_files": "ファイル",
        "ui.tab_shell": "シェル",
        "ui.tab_ssh": "SSH ターゲット",
        "ui.tab_settings": "設定",
        "ui.system": "システム",
        "ui.hostname": "ホスト名",
        "ui.os": "OS",
        "ui.python": "Python",
        "ui.uptime": "稼働時間",
        "ui.cpu_usage": "CPU 使用率",
        "ui.cores": "コア",
        "ui.ram": "RAM",
        "ui.bind_port": "バインド / ポート",
        "ui.storage": "ストレージ",
        "ui.edit_selection": "選択を編集",
        "ui.close_selection": "選択を閉じる",
        "ui.disk_hint": "既定では 5 GiB 以上のメディアのみ表示されます。小さいメディアはここでチェックすると表示されます。",
        "ui.save_selection": "選択を保存",
        "ui.select_all": "すべて選択",
        "ui.select_none": "すべて解除",
        "ui.default_only_big": "既定 (≥ 5 GiB のみ)",
        "ui.cancel": "キャンセル",
        "ui.no_disks": "ストレージメディアが選択されていないか、見つかりません。",
        "ui.disk": "ディスク",
        "ui.small": "小",
        "ui.open_in_files": "ファイルで開く",
        "ui.auto_ge": "自動 ≥ {size}",
        "ui.manual": "手動",
        "ui.shown": "表示中",
        "ui.load_avg": "平均負荷 (1 / 5 / 15 分)",
        "ui.default_saved": "既定を保存しました",
        "ui.selection_saved": "選択を保存しました",
        "ui.default_restored": "既定に戻しました",
        "ui.filter_cmd": "名前またはコマンドで絞り込み…",
        "ui.refresh": "更新",
        "ui.pid": "PID",
        "ui.name": "名前",
        "ui.user": "ユーザー",
        "ui.cpu_pct": "CPU %",
        "ui.ram_pct": "RAM %",
        "ui.cmdline": "コマンドライン",
        "ui.no_procs": "プロセスがありません。",
        "ui.kill": "終了",
        "ui.kill_confirm_title": "PID {pid} を終了しますか?",
        "ui.kill_confirm_msg": "SIGTERM を送信します。",
        "ui.kill_ok_label": "終了",
        "ui.killed": "PID {pid} を終了しました",
        "ui.filter": "絞り込み…",
        "ui.load": "負荷",
        "ui.active": "状態",
        "ui.sub": "サブ",
        "ui.no_services": "サービスがありません。",
        "ui.start": "開始",
        "ui.stop": "停止",
        "ui.restart": "再起動",
        "ui.logs": "ログ",
        "ui.logs_title": "ログ: {name}",
        "ui.close": "閉じる",
        "ui.path": "/path/to/folder",
        "ui.parent": "↑ 親",
        "ui.open": "開く",
        "ui.sudo_missing": "この SSH ターゲットには sudo パスワードがありません。保護されたフォルダへの書き込みは失敗する可能性があります。",
        "ui.sudo_ok": "Sudo 有効 — 必要に応じて sudo で書き込みます。",
        "ui.upload_title": "ファイルをアップロード",
        "ui.upload_hint": "クリックまたはここにドラッグ。権限がない場合は sudo を使用します。",
        "ui.target_dir": "現在の対象フォルダ:",
        "ui.selected_n": "{n} 件選択中",
        "ui.select_all_btn": "すべて選択",
        "ui.clear_selection": "選択をクリア",
        "ui.zip_selection": "選択を ZIP に",
        "ui.new_folder": "新しいフォルダ",
        "ui.delete_selection": "選択を削除",
        "ui.perm": "権限",
        "ui.owner": "所有者",
        "ui.size": "サイズ",
        "ui.modified": "更新日時",
        "ui.empty": "空です。",
        "ui.edit": "編集",
        "ui.download": "ダウンロード",
        "ui.delete": "削除",
        "ui.rename": "名前変更",
        "ui.perms": "権限",
        "ui.chown": "所有者",
        "ui.save": "保存",
        "ui.saved": "保存しました",
        "ui.save_failed": "保存に失敗しました",
        "ui.delete_confirm": "{n} 件を削除しますか?",
        "ui.delete_recursive": "フォルダは再帰的に削除されます。",
        "ui.delete_ok_label": "削除",
        "ui.deleted_n": "{n} 件を削除しました",
        "ui.deleted_failed": "{total} 件中 {failed} 件が失敗しました",
        "ui.deleted_one": "削除しました",
        "ui.delete_error": "削除に失敗しました",
        "ui.rename_title": "名前を変更",
        "ui.rename_prompt": "\"{name}\" の新しい名前:",
        "ui.rename_ok_label": "変更",
        "ui.renamed": "変更しました",
        "ui.perms_title": "権限を設定",
        "ui.perms_ok_label": "設定",
        "ui.invalid_mode": "無効なモード",
        "ui.perms_set": "権限を設定しました",
        "ui.chown_title": "所有者を変更",
        "ui.chown_prompt": "\"{path}\" の user[:group]:",
        "ui.owner_set": "所有者を設定しました",
        "ui.new_folder_title": "{dir} に新しいフォルダ",
        "ui.new_folder_prompt": "名前:",
        "ui.create_label": "作成",
        "ui.folder_created": "フォルダを作成しました",
        "ui.zip_created": "ZIP を作成しました",
        "ui.no_selection": "選択されていません",
        "ui.upload_failed": "{name} のエラー: {err}",
        "ui.upload_processing": "サーバー処理中…",
        "ui.upload_via_sudo": "sudo 経由",
        "ui.upload_total": "合計",
        "ui.upload_error": "エラー:",
        "ui.loading": "読み込み中…",
        "ui.shell_placeholder": "コマンドを入力… (例: uptime)",
        "ui.with_sudo": "sudo を使用",
        "ui.run": "実行",
        "ui.ready": "準備完了。",
        "ui.shell_running": "{target} で実行中:",
        "ui.ssh_warning": "SSH パスワード、sudo パスワード、キーのパスフレーズは config.json に平文で保存されます。ファイルを適切に保護してください (chmod 600)。",
        "ui.add_edit_ssh": "SSH ターゲットを追加 / 編集",
        "ui.host_ip": "ホスト / IP",
        "ui.auth_method": "認証方式",
        "ui.auth_password": "パスワード",
        "ui.auth_key": "SSH キー",
        "ui.ssh_password": "SSH パスワード (保存されます)",
        "ui.ssh_key_path": "秘密鍵のパス",
        "ui.ssh_key_pass": "キーのパスフレーズ (任意)",
        "ui.ssh_key_pass_ph": "暗号化されていない場合は空",
        "ui.sudo_password": "sudo パスワード (任意 — /etc, /var などへの書き込み用)",
        "ui.sudo_password_ph": "sudo が不要な場合は空",
        "ui.clear_sudo": "sudo パスワードを削除",
        "ui.save_test": "保存してテスト",
        "ui.clear_form": "フォームをクリア",
        "ui.saved_ssh": "保存しました – {msg}",
        "ui.stored_targets": "保存された SSH ターゲット",
        "ui.port": "ポート",
        "ui.auth": "認証",
        "ui.sudo": "Sudo",
        "ui.yes": "有効",
        "ui.no": "いいえ",
        "ui.test": "テスト",
        "ui.edit_btn": "編集",
        "ui.sudo_password_btn": "sudo パスワード",
        "ui.delete_btn": "削除",
        "ui.no_targets": "SSH ターゲットが保存されていません。",
        "ui.target_deleted": "削除しました",
        "ui.target_delete_title": "SSH ターゲットを削除しますか?",
        "ui.form_loaded": "フォームを読み込みました。",
        "ui.fields_required": "名前・ホスト・ユーザーは必須です",
        "ui.sudo_pw_title": "sudo パスワード",
        "ui.sudo_pw_prompt": "\"{name}\" のパスワード (保存されます):",
        "ui.set": "設定",
        "ui.current_config": "現在の構成",
        "ui.bind_addr_port": "バインドアドレスとポート",
        "ui.bind_ip": "バインド IP",
        "ui.custom_ip": "または独自の IP を入力",
        "ui.custom_ip_ph": "例: 192.168.1.100",
        "ui.random_port": "ランダムポート",
        "ui.check_port": "ポートを確認",
        "ui.change_login": "ログインを変更 (任意)",
        "ui.new_user_ph": "新しいユーザー名",
        "ui.new_pass_ph": "新しいパスワード",
        "ui.new_pass2_ph": "パスワードを再入力",
        "ui.allow_delete": "ファイルブラウザでの削除を許可",
        "ui.reset": "リセット",
        "ui.restart_note": "保存後: CTRL+C でサーバーを停止し、再度起動してください。",
        "ui.port_random": "ランダムポートを選択しました。",
        "ui.port_enter": "ポートを入力してください。",
        "ui.checking": "確認中…",
        "ui.password_mismatch": "パスワードが一致しません",
        "ui.no_port": "ポートが指定されていません",
        "ui.config_saved": "保存しました。\n再起動後:\n{url}",
        "ui.error": "エラー: {err}",
        "ui.language": "言語",
    },
    "ko": {
        "setup.title": "PC Overlay - 초기 설정",
        "setup.config_path": "설정이 저장될 위치:",
        "setup.available_ips": "사용 가능한 IP:",
        "setup.recommended": "(LAN 접속에 권장)",
        "setup.bind_ip": "바인드 IP",
        "setup.port": "포트",
        "setup.username": "사용자 이름",
        "setup.password": "비밀번호",
        "setup.password_repeat": "비밀번호 다시 입력",
        "setup.invalid_ip": "잘못된 IP '{ip}', 0.0.0.0 사용",
        "setup.invalid_port": "포트는 1024~65535 사이여야 합니다.",
        "setup.reserved_port": "포트 {port} 은(는) 잘 알려진 서비스용으로 예약되어 있습니다.",
        "setup.port_busy": "포트 {port} 은(는) {host} 에서 사용 중입니다.",
        "setup.not_a_number": "숫자가 아닙니다.",
        "setup.password_empty": "비밀번호는 비워둘 수 없습니다.",
        "setup.password_mismatch": "비밀번호가 일치하지 않습니다.",
        "setup.done_title": "설정 완료",
        "setup.done_config": "설정:",
        "setup.done_bind": "바인드:",
        "setup.done_user": "사용자:",
        "banner.running": "{app} 실행 중",
        "banner.bind": "바인드:",
        "banner.local": "로컬:",
        "banner.network": "네트워크:",
        "banner.user": "사용자:",
        "banner.password": "비밀번호: (config.json 에 해시만 저장)",
        "banner.ssh_targets": "저장된 SSH 대상:",
        "banner.disk_display": "디스크 표시: {gb} GiB 이상만 자동 표시",
        "banner.lan_only": "로컬 네트워크에서만 접근 가능합니다.",
        "banner.quit": "CTRL+C 로 종료합니다.",
        "banner.stopped": "종료되었습니다.",
        "banner.bind_error": "오류: {ip}:{port} 에 바인드할 수 없습니다.",
        "banner.config_deleted": "설정 삭제됨:",
        "banner.config_unreadable": "config.json 을(를) 읽을 수 없습니다:",
        "err.auth": "사용자 이름 또는 비밀번호가 잘못되었습니다",
        "err.network": "{ip} 에 대한 접근이 거부되었습니다. LAN/VPN 만 허용.",
        "err.overlay_not_found": "오버레이 '{name}' 을(를) 찾을 수 없습니다",
        "err.no_ssh_target": "'{name}' 이라는 SSH 대상이 없습니다",
        "err.path_not_found": "경로를 찾을 수 없습니다",
        "err.no_access": "접근 권한 없음",
        "err.no_file": "파일이 아닙니다",
        "err.file_too_big": "브라우저에서 편집하기에는 파일이 너무 큽니다",
        "err.path_missing": "경로가 없습니다",
        "err.no_write": "쓰기 권한이 없습니다",
        "err.no_write_dir": "대상 폴더에 쓰기 권한이 없습니다.",
        "err.save_failed": "저장 실패",
        "err.file_not_found": "파일을 찾을 수 없습니다",
        "err.no_paths": "경로가 지정되지 않았습니다",
        "err.zip_failed": "ZIP 생성 실패: {err}",
        "err.invalid_action": "잘못된 작업",
        "err.invalid_service": "잘못된 서비스 이름",
        "err.no_cmd": "명령이 없습니다",
        "err.cmd_not_allowed": "허용되지 않은 명령입니다",
        "err.delete_disabled": "삭제가 비활성화되어 있습니다",
        "err.name_required": "경로와 새 이름이 필요합니다",
        "err.invalid_name": "잘못된 이름",
        "err.src_not_found": "원본을 찾을 수 없습니다",
        "err.dst_exists": "대상이 이미 존재합니다",
        "err.mv_failed": "mv 실패",
        "err.mkdir_failed": "mkdir 실패",
        "err.mode_octal": "모드는 8진수여야 합니다 (예: 644)",
        "err.invalid_owner": "잘못된 소유자 (user[:group])",
        "err.chmod_failed": "chmod 실패",
        "err.chown_failed": "chown 실패",
        "err.local_reserved": "'local' 은 예약어입니다",
        "err.fields_required": "이름, 호스트, 사용자는 필수입니다",
        "err.invalid_port": "잘못된 포트",
        "err.auth_type": "auth 는 'password' 또는 'key' 여야 합니다",
        "err.not_found": "찾을 수 없습니다",
        "err.password_empty": "비밀번호는 비워둘 수 없습니다",
        "err.invalid_bind": "잘못된 바인드 IP: {ip}",
        "err.port_not_allowed": "포트 {port} 은(는) 허용되지 않습니다.",
        "err.port_busy": "포트 {port} 은(는) 사용 중입니다.",
        "err.no_valid_port": "유효한 포트가 아닙니다",
        "err.port_out_of_range": "포트가 1024~65535 범위를 벗어났습니다.",
        "err.port_reserved": "포트 {port} 은(는) 예약되어 있습니다.",
        "err.target_dir_invalid": "대상 폴더가 존재하지 않거나 디렉터리가 아닙니다",
        "err.write_failed": "쓰기 실패: {err}",
        "err.no_write_sudo": "{path} 에 대한 쓰기 권한이 없습니다. sudo 비밀번호를 설정하세요.",
        "err.tmp_write": "tmp 쓰기 실패: {err}",
        "err.mv_failed_sudo": "mv 실패: {err}",
        "err.sudo_upload": "sudo 업로드 실패: {err}",
        "err.timeout": "{s}초 후 시간 초과",
        "ok.conn": "연결 성공",
        "ok.conn_failed": "연결 실패: {err}",
        "ok.sudo_works": "sudo 작동",
        "ok.sudo_failed": "sudo 테스트 실패: {err}",
        "ok.port_free": "포트 {port} 은(는) 비어 있고 허용됩니다.",
        "ok.saved_restart": "저장했습니다. 변경 사항을 적용하려면 서버를 다시 시작하세요.",
        "ok.test_failed": "실패: {err}",
        "ui.overlay": "오버레이",
        "ui.overlay_local": "이 컴퓨터에서",
        "ui.tab_dash": "대시보드",
        "ui.tab_proc": "프로세스",
        "ui.tab_svc": "서비스",
        "ui.tab_files": "파일",
        "ui.tab_shell": "셸",
        "ui.tab_ssh": "SSH 대상",
        "ui.tab_settings": "설정",
        "ui.system": "시스템",
        "ui.hostname": "호스트 이름",
        "ui.os": "OS",
        "ui.python": "Python",
        "ui.uptime": "가동 시간",
        "ui.cpu_usage": "CPU 사용량",
        "ui.cores": "코어",
        "ui.ram": "RAM",
        "ui.bind_port": "바인드 / 포트",
        "ui.storage": "저장소",
        "ui.edit_selection": "선택 편집",
        "ui.close_selection": "선택 닫기",
        "ui.disk_hint": "기본적으로 5 GiB 이상의 미디어만 표시됩니다. 더 작은 미디어는 여기에서 선택하면 표시됩니다.",
        "ui.save_selection": "선택 저장",
        "ui.select_all": "모두 선택",
        "ui.select_none": "선택 해제",
        "ui.default_only_big": "기본 (≥ 5 GiB 만)",
        "ui.cancel": "취소",
        "ui.no_disks": "선택되었거나 발견된 저장소 미디어가 없습니다.",
        "ui.disk": "디스크",
        "ui.small": "작음",
        "ui.open_in_files": "파일에서 열기",
        "ui.auto_ge": "자동 ≥ {size}",
        "ui.manual": "수동",
        "ui.shown": "표시 중",
        "ui.load_avg": "평균 부하 (1 / 5 / 15 분)",
        "ui.default_saved": "기본값 저장됨",
        "ui.selection_saved": "선택 저장됨",
        "ui.default_restored": "기본값 복원됨",
        "ui.filter_cmd": "이름 또는 명령으로 필터…",
        "ui.refresh": "새로 고침",
        "ui.pid": "PID",
        "ui.name": "이름",
        "ui.user": "사용자",
        "ui.cpu_pct": "CPU %",
        "ui.ram_pct": "RAM %",
        "ui.cmdline": "명령줄",
        "ui.no_procs": "프로세스가 없습니다.",
        "ui.kill": "종료",
        "ui.kill_confirm_title": "PID {pid} 을(를) 종료할까요?",
        "ui.kill_confirm_msg": "SIGTERM 을 보냅니다.",
        "ui.kill_ok_label": "종료",
        "ui.killed": "PID {pid} 종료됨",
        "ui.filter": "필터…",
        "ui.load": "부하",
        "ui.active": "상태",
        "ui.sub": "하위",
        "ui.no_services": "서비스가 없습니다.",
        "ui.start": "시작",
        "ui.stop": "중지",
        "ui.restart": "재시작",
        "ui.logs": "로그",
        "ui.logs_title": "로그: {name}",
        "ui.close": "닫기",
        "ui.path": "/path/to/folder",
        "ui.parent": "↑ 상위",
        "ui.open": "열기",
        "ui.sudo_missing": "이 SSH 대상에는 sudo 비밀번호가 없습니다. 보호된 폴더에 대한 쓰기가 실패할 수 있습니다.",
        "ui.sudo_ok": "Sudo 활성 — 필요 시 sudo 로 쓰기를 수행합니다.",
        "ui.upload_title": "파일 업로드",
        "ui.upload_hint": "클릭하거나 파일을 여기로 끌어오세요. 권한이 없으면 sudo 를 사용합니다.",
        "ui.target_dir": "현재 대상 폴더:",
        "ui.selected_n": "{n} 개 선택됨",
        "ui.select_all_btn": "모두 선택",
        "ui.clear_selection": "선택 지우기",
        "ui.zip_selection": "선택 항목을 ZIP 으로",
        "ui.new_folder": "새 폴더",
        "ui.delete_selection": "선택 삭제",
        "ui.perm": "권한",
        "ui.owner": "소유자",
        "ui.size": "크기",
        "ui.modified": "수정됨",
        "ui.empty": "비어 있음.",
        "ui.edit": "편집",
        "ui.download": "다운로드",
        "ui.delete": "삭제",
        "ui.rename": "이름 변경",
        "ui.perms": "권한",
        "ui.chown": "소유자",
        "ui.save": "저장",
        "ui.saved": "저장됨",
        "ui.save_failed": "저장 실패",
        "ui.delete_confirm": "{n} 개 항목을 삭제할까요?",
        "ui.delete_recursive": "폴더는 재귀적으로 삭제됩니다.",
        "ui.delete_ok_label": "삭제",
        "ui.deleted_n": "{n} 개 삭제됨",
        "ui.deleted_failed": "{total} 개 중 {failed} 개 실패",
        "ui.deleted_one": "삭제됨",
        "ui.delete_error": "삭제 실패",
        "ui.rename_title": "이름 변경",
        "ui.rename_prompt": "\"{name}\" 의 새 이름:",
        "ui.rename_ok_label": "변경",
        "ui.renamed": "변경됨",
        "ui.perms_title": "권한 설정",
        "ui.perms_ok_label": "설정",
        "ui.invalid_mode": "잘못된 모드",
        "ui.perms_set": "권한 설정됨",
        "ui.chown_title": "소유자 변경",
        "ui.chown_prompt": "\"{path}\" 의 user[:group]:",
        "ui.owner_set": "소유자 설정됨",
        "ui.new_folder_title": "{dir} 에 새 폴더",
        "ui.new_folder_prompt": "이름:",
        "ui.create_label": "만들기",
        "ui.folder_created": "폴더 생성됨",
        "ui.zip_created": "ZIP 생성됨",
        "ui.no_selection": "선택 없음",
        "ui.upload_failed": "{name} 오류: {err}",
        "ui.upload_processing": "서버 처리 중…",
        "ui.upload_via_sudo": "sudo 통해",
        "ui.upload_total": "합계",
        "ui.upload_error": "오류:",
        "ui.loading": "로드 중…",
        "ui.shell_placeholder": "명령 입력… (예: uptime)",
        "ui.with_sudo": "sudo 사용",
        "ui.run": "실행",
        "ui.ready": "준비됨.",
        "ui.shell_running": "{target} 에서 실행 중:",
        "ui.ssh_warning": "SSH 비밀번호, sudo 비밀번호 및 키 암호는 config.json 에 평문으로 저장됩니다. 파일을 적절히 보호하세요 (chmod 600).",
        "ui.add_edit_ssh": "SSH 대상 추가 / 편집",
        "ui.host_ip": "호스트 / IP",
        "ui.auth_method": "인증 방식",
        "ui.auth_password": "비밀번호",
        "ui.auth_key": "SSH 키",
        "ui.ssh_password": "SSH 비밀번호 (저장됨)",
        "ui.ssh_key_path": "개인 키 경로",
        "ui.ssh_key_pass": "키 암호 (선택)",
        "ui.ssh_key_pass_ph": "암호화되지 않은 경우 비워둠",
        "ui.sudo_password": "sudo 비밀번호 (선택 — /etc, /var 등에 쓰기용)",
        "ui.sudo_password_ph": "sudo 가 필요 없으면 비워둠",
        "ui.clear_sudo": "sudo 비밀번호 삭제",
        "ui.save_test": "저장 및 테스트",
        "ui.clear_form": "양식 지우기",
        "ui.saved_ssh": "저장됨 – {msg}",
        "ui.stored_targets": "저장된 SSH 대상",
        "ui.port": "포트",
        "ui.auth": "인증",
        "ui.sudo": "Sudo",
        "ui.yes": "활성",
        "ui.no": "아니오",
        "ui.test": "테스트",
        "ui.edit_btn": "편집",
        "ui.sudo_password_btn": "sudo 비밀번호",
        "ui.delete_btn": "삭제",
        "ui.no_targets": "저장된 SSH 대상이 없습니다.",
        "ui.target_deleted": "삭제됨",
        "ui.target_delete_title": "SSH 대상을 삭제할까요?",
        "ui.form_loaded": "양식 로드됨.",
        "ui.fields_required": "이름, 호스트, 사용자는 필수입니다",
        "ui.sudo_pw_title": "sudo 비밀번호",
        "ui.sudo_pw_prompt": "\"{name}\" 의 비밀번호 (저장됨):",
        "ui.set": "설정",
        "ui.current_config": "현재 구성",
        "ui.bind_addr_port": "바인드 주소 및 포트",
        "ui.bind_ip": "바인드 IP",
        "ui.custom_ip": "또는 사용자 지정 IP 입력",
        "ui.custom_ip_ph": "예: 192.168.1.100",
        "ui.random_port": "임의 포트",
        "ui.check_port": "포트 확인",
        "ui.change_login": "로그인 변경 (선택)",
        "ui.new_user_ph": "새 사용자 이름",
        "ui.new_pass_ph": "새 비밀번호",
        "ui.new_pass2_ph": "비밀번호 다시 입력",
        "ui.allow_delete": "파일 브라우저에서 삭제 허용",
        "ui.reset": "초기화",
        "ui.restart_note": "저장 후: CTRL+C 로 서버를 중지하고 다시 시작하세요.",
        "ui.port_random": "임의 포트를 선택했습니다.",
        "ui.port_enter": "포트를 입력하세요.",
        "ui.checking": "확인 중…",
        "ui.password_mismatch": "비밀번호가 일치하지 않습니다",
        "ui.no_port": "포트가 지정되지 않았습니다",
        "ui.config_saved": "저장됨.\n재시작 후:\n{url}",
        "ui.error": "오류: {err}",
        "ui.language": "언어",
    },
    "zh": {
        "setup.title": "PC Overlay - 初始设置",
        "setup.config_path": "配置将保存到：",
        "setup.available_ips": "可用 IP：",
        "setup.recommended": "（推荐用于局域网访问）",
        "setup.bind_ip": "绑定 IP",
        "setup.port": "端口",
        "setup.username": "用户名",
        "setup.password": "密码",
        "setup.password_repeat": "重复密码",
        "setup.invalid_ip": "无效 IP '{ip}'，使用 0.0.0.0",
        "setup.invalid_port": "端口必须在 1024 到 65535 之间。",
        "setup.reserved_port": "端口 {port} 为已知服务保留。",
        "setup.port_busy": "端口 {port} 在 {host} 上已被占用。",
        "setup.not_a_number": "不是数字。",
        "setup.password_empty": "密码不能为空。",
        "setup.password_mismatch": "密码不匹配。",
        "setup.done_title": "设置完成",
        "setup.done_config": "配置：",
        "setup.done_bind": "绑定：",
        "setup.done_user": "用户：",
        "banner.running": "{app} 正在运行",
        "banner.bind": "绑定：",
        "banner.local": "本地：",
        "banner.network": "网络：",
        "banner.user": "用户：",
        "banner.password": "密码：（仅哈希，保存在 config.json）",
        "banner.ssh_targets": "已保存的 SSH 目标：",
        "banner.disk_display": "磁盘显示：仅自动显示 ≥ {gb} GiB 的介质",
        "banner.lan_only": "仅限本地网络访问。",
        "banner.quit": "按 CTRL+C 退出。",
        "banner.stopped": "已停止。",
        "banner.bind_error": "错误：无法绑定到 {ip}:{port}。",
        "banner.config_deleted": "已删除配置：",
        "banner.config_unreadable": "无法读取 config.json：",
        "err.auth": "用户名或密码错误",
        "err.network": "拒绝访问 {ip}。仅允许 LAN/VPN。",
        "err.overlay_not_found": "找不到覆盖层 '{name}'",
        "err.no_ssh_target": "没有名为 '{name}' 的 SSH 目标",
        "err.path_not_found": "路径不存在",
        "err.no_access": "无访问权限",
        "err.no_file": "不是文件",
        "err.file_too_big": "文件太大，无法在浏览器中编辑",
        "err.path_missing": "缺少路径",
        "err.no_write": "无写入权限",
        "err.no_write_dir": "目标文件夹无写入权限。",
        "err.save_failed": "保存失败",
        "err.file_not_found": "找不到文件",
        "err.no_paths": "未提供路径",
        "err.zip_failed": "创建 ZIP 失败：{err}",
        "err.invalid_action": "无效操作",
        "err.invalid_service": "无效的服务名称",
        "err.no_cmd": "无命令",
        "err.cmd_not_allowed": "命令不被允许",
        "err.delete_disabled": "删除已被禁用",
        "err.name_required": "需要路径和新名称",
        "err.invalid_name": "无效名称",
        "err.src_not_found": "找不到源",
        "err.dst_exists": "目标已存在",
        "err.mv_failed": "mv 失败",
        "err.mkdir_failed": "mkdir 失败",
        "err.mode_octal": "模式必须为八进制（例如 644）",
        "err.invalid_owner": "无效的所有者（user[:group]）",
        "err.chmod_failed": "chmod 失败",
        "err.chown_failed": "chown 失败",
        "err.local_reserved": "'local' 是保留名称",
        "err.fields_required": "名称、主机和用户为必填项",
        "err.invalid_port": "无效端口",
        "err.auth_type": "auth 必须为 'password' 或 'key'",
        "err.not_found": "未找到",
        "err.password_empty": "密码不能为空",
        "err.invalid_bind": "无效的绑定 IP：{ip}",
        "err.port_not_allowed": "端口 {port} 不被允许。",
        "err.port_busy": "端口 {port} 已被占用。",
        "err.no_valid_port": "不是有效端口",
        "err.port_out_of_range": "端口超出 1024–65535。",
        "err.port_reserved": "端口 {port} 已保留。",
        "err.target_dir_invalid": "目标文件夹不存在或不是目录",
        "err.write_failed": "写入失败：{err}",
        "err.no_write_sudo": "对 {path} 无写入权限。请设置 sudo 密码。",
        "err.tmp_write": "tmp 写入失败：{err}",
        "err.mv_failed_sudo": "mv 失败：{err}",
        "err.sudo_upload": "sudo 上传失败：{err}",
        "err.timeout": "{s} 秒后超时",
        "ok.conn": "连接成功",
        "ok.conn_failed": "连接失败：{err}",
        "ok.sudo_works": "sudo 正常工作",
        "ok.sudo_failed": "sudo 测试失败：{err}",
        "ok.port_free": "端口 {port} 空闲且允许使用。",
        "ok.saved_restart": "已保存。请重启服务器以应用更改。",
        "ok.test_failed": "失败：{err}",
        "ui.overlay": "覆盖层",
        "ui.overlay_local": "本机",
        "ui.tab_dash": "仪表盘",
        "ui.tab_proc": "进程",
        "ui.tab_svc": "服务",
        "ui.tab_files": "文件",
        "ui.tab_shell": "Shell",
        "ui.tab_ssh": "SSH 目标",
        "ui.tab_settings": "设置",
        "ui.system": "系统",
        "ui.hostname": "主机名",
        "ui.os": "操作系统",
        "ui.python": "Python",
        "ui.uptime": "运行时间",
        "ui.cpu_usage": "CPU 使用率",
        "ui.cores": "核心",
        "ui.ram": "内存",
        "ui.bind_port": "绑定 / 端口",
        "ui.storage": "存储",
        "ui.edit_selection": "编辑选择",
        "ui.close_selection": "关闭选择",
        "ui.disk_hint": "默认仅显示 5 GiB 及以上的介质。更小的介质在勾选后显示。",
        "ui.save_selection": "保存选择",
        "ui.select_all": "全选",
        "ui.select_none": "全不选",
        "ui.default_only_big": "默认（仅 ≥ 5 GiB）",
        "ui.cancel": "取消",
        "ui.no_disks": "未选择或未找到存储介质。",
        "ui.disk": "磁盘",
        "ui.small": "小",
        "ui.open_in_files": "在文件中打开",
        "ui.auto_ge": "自动 ≥ {size}",
        "ui.manual": "手动",
        "ui.shown": "已显示",
        "ui.load_avg": "平均负载（1 / 5 / 15 分钟）",
        "ui.default_saved": "默认设置已保存",
        "ui.selection_saved": "选择已保存",
        "ui.default_restored": "已恢复默认",
        "ui.filter_cmd": "按名称或命令过滤…",
        "ui.refresh": "刷新",
        "ui.pid": "PID",
        "ui.name": "名称",
        "ui.user": "用户",
        "ui.cpu_pct": "CPU %",
        "ui.ram_pct": "内存 %",
        "ui.cmdline": "命令行",
        "ui.no_procs": "无进程。",
        "ui.kill": "结束",
        "ui.kill_confirm_title": "结束 PID {pid}？",
        "ui.kill_confirm_msg": "将发送 SIGTERM。",
        "ui.kill_ok_label": "结束",
        "ui.killed": "已结束 PID {pid}",
        "ui.filter": "过滤…",
        "ui.load": "负载",
        "ui.active": "状态",
        "ui.sub": "子状态",
        "ui.no_services": "无服务。",
        "ui.start": "启动",
        "ui.stop": "停止",
        "ui.restart": "重启",
        "ui.logs": "日志",
        "ui.logs_title": "日志：{name}",
        "ui.close": "关闭",
        "ui.path": "/路径/到/文件夹",
        "ui.parent": "↑ 上级",
        "ui.open": "打开",
        "ui.sudo_missing": "此 SSH 目标没有 sudo 密码。写入受保护文件夹可能失败。",
        "ui.sudo_ok": "Sudo 已启用 — 需要时将通过 sudo 执行写入。",
        "ui.upload_title": "上传文件",
        "ui.upload_hint": "点击或将文件拖到此处。权限不足时使用 sudo。",
        "ui.target_dir": "当前目标文件夹：",
        "ui.selected_n": "已选择 {n} 项",
        "ui.select_all_btn": "全选",
        "ui.clear_selection": "清除选择",
        "ui.zip_selection": "将选择打包为 ZIP",
        "ui.new_folder": "新建文件夹",
        "ui.delete_selection": "删除选择",
        "ui.perm": "权限",
        "ui.owner": "所有者",
        "ui.size": "大小",
        "ui.modified": "修改时间",
        "ui.empty": "空。",
        "ui.edit": "编辑",
        "ui.download": "下载",
        "ui.delete": "删除",
        "ui.rename": "重命名",
        "ui.perms": "权限",
        "ui.chown": "所有者",
        "ui.save": "保存",
        "ui.saved": "已保存",
        "ui.save_failed": "保存失败",
        "ui.delete_confirm": "删除 {n} 项？",
        "ui.delete_recursive": "文件夹将被递归删除。",
        "ui.delete_ok_label": "删除",
        "ui.deleted_n": "已删除 {n} 项",
        "ui.deleted_failed": "{total} 项中 {failed} 项失败",
        "ui.deleted_one": "已删除",
        "ui.delete_error": "删除失败",
        "ui.rename_title": "重命名",
        "ui.rename_prompt": "\"{name}\" 的新名称：",
        "ui.rename_ok_label": "重命名",
        "ui.renamed": "已重命名",
        "ui.perms_title": "设置权限",
        "ui.perms_ok_label": "设置",
        "ui.invalid_mode": "无效模式",
        "ui.perms_set": "已设置权限",
        "ui.chown_title": "更改所有者",
        "ui.chown_prompt": "\"{path}\" 的 user[:group]：",
        "ui.owner_set": "已设置所有者",
        "ui.new_folder_title": "在 {dir} 中新建文件夹",
        "ui.new_folder_prompt": "名称：",
        "ui.create_label": "创建",
        "ui.folder_created": "已创建文件夹",
        "ui.zip_created": "已创建 ZIP",
        "ui.no_selection": "未选择任何内容",
        "ui.upload_failed": "{name} 出错：{err}",
        "ui.upload_processing": "服务器处理中…",
        "ui.upload_via_sudo": "通过 sudo",
        "ui.upload_total": "总计",
        "ui.upload_error": "错误：",
        "ui.loading": "加载中…",
        "ui.shell_placeholder": "输入命令…（例如 uptime）",
        "ui.with_sudo": "使用 sudo",
        "ui.run": "运行",
        "ui.ready": "就绪。",
        "ui.shell_running": "正在 {target} 上运行：",
        "ui.ssh_warning": "SSH 密码、sudo 密码和密钥口令以明文形式存储在 config.json 中。请妥善保护该文件（chmod 600）。",
        "ui.add_edit_ssh": "添加 / 编辑 SSH 目标",
        "ui.host_ip": "主机 / IP",
        "ui.auth_method": "认证方式",
        "ui.auth_password": "密码",
        "ui.auth_key": "SSH 密钥",
        "ui.ssh_password": "SSH 密码（将被保存）",
        "ui.ssh_key_path": "私钥路径",
        "ui.ssh_key_pass": "密钥口令（可选）",
        "ui.ssh_key_pass_ph": "未加密则留空",
        "ui.sudo_password": "sudo 密码（可选 — 用于写入 /etc、/var 等）",
        "ui.sudo_password_ph": "无需 sudo 时留空",
        "ui.clear_sudo": "删除 sudo 密码",
        "ui.save_test": "保存并测试",
        "ui.clear_form": "清空表单",
        "ui.saved_ssh": "已保存 – {msg}",
        "ui.stored_targets": "已保存的 SSH 目标",
        "ui.port": "端口",
        "ui.auth": "认证",
        "ui.sudo": "Sudo",
        "ui.yes": "已启用",
        "ui.no": "否",
        "ui.test": "测试",
        "ui.edit_btn": "编辑",
        "ui.sudo_password_btn": "sudo 密码",
        "ui.delete_btn": "删除",
        "ui.no_targets": "未保存任何 SSH 目标。",
        "ui.target_deleted": "已删除",
        "ui.target_delete_title": "删除 SSH 目标？",
        "ui.form_loaded": "已加载表单。",
        "ui.fields_required": "名称、主机和用户为必填项",
        "ui.sudo_pw_title": "sudo 密码",
        "ui.sudo_pw_prompt": "\"{name}\" 的密码（将被保存）：",
        "ui.set": "设置",
        "ui.current_config": "当前配置",
        "ui.bind_addr_port": "绑定地址和端口",
        "ui.bind_ip": "绑定 IP",
        "ui.custom_ip": "或输入自定义 IP",
        "ui.custom_ip_ph": "例如 192.168.1.100",
        "ui.random_port": "随机端口",
        "ui.check_port": "检查端口",
        "ui.change_login": "更改登录（可选）",
        "ui.new_user_ph": "新用户名",
        "ui.new_pass_ph": "新密码",
        "ui.new_pass2_ph": "重复密码",
        "ui.allow_delete": "允许在文件浏览器中删除",
        "ui.reset": "重置",
        "ui.restart_note": "保存后：使用 CTRL+C 停止服务器并重新启动。",
        "ui.port_random": "已选择随机端口。",
        "ui.port_enter": "请输入端口。",
        "ui.checking": "检查中…",
        "ui.password_mismatch": "密码不匹配",
        "ui.no_port": "未指定端口",
        "ui.config_saved": "已保存。\n重启后：\n{url}",
        "ui.error": "错误：{err}",
        "ui.language": "语言",
    },
}


def t(key: str, lang: str | None = None, **kwargs) -> str:
    """Übersetzt einen Key in die aktive Sprache (Fallback: Englisch, dann Key selbst)."""
    active = (lang or _ACTIVE_LANG or DEFAULT_LANG)
    table = LOCALES.get(active) or LOCALES[DEFAULT_LANG]
    s = table.get(key)
    if s is None:
        s = LOCALES[DEFAULT_LANG].get(key, key)
    if kwargs:
        try:
            return s.format(**kwargs)
        except (KeyError, IndexError):
            return s
    return s


# Globale aktive Sprache (wird in main() gesetzt)
_ACTIVE_LANG: str = DEFAULT_LANG


# =============================================================
# HILFSFUNKTIONEN
# =============================================================
def random_exotic_port() -> int:
    while True:
        p = secrets.randbelow(PORT_RANDOM_MAX - PORT_RANDOM_MIN) + PORT_RANDOM_MIN
        if p not in FORBIDDEN_PORTS:
            return p


def port_is_allowed(port: int) -> bool:
    return isinstance(port, int) and 1024 <= port <= 65535 and port not in FORBIDDEN_PORTS


def get_lan_ip() -> str:
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    try:
        s.connect(("8.8.8.8", 80))
        return s.getsockname()[0]
    except Exception:
        return "127.0.0.1"
    finally:
        s.close()


def list_all_ips() -> list[dict]:
    out: list[dict] = []
    try:
        for iface, addrs in psutil.net_if_addrs().items():
            for a in addrs:
                if a.family == socket.AF_INET:
                    out.append({"iface": iface, "ip": a.address, "family": "IPv4"})
                elif a.family == socket.AF_INET6 and "%" not in a.address:
                    out.append({"iface": iface, "ip": a.address, "family": "IPv6"})
    except Exception:
        pass
    seen, uniq = set(), []
    for item in out:
        k = (item["iface"], item["ip"])
        if k not in seen:
            seen.add(k)
            uniq.append(item)
    uniq.insert(0, {"iface": "alle", "ip": "0.0.0.0", "family": "IPv4"})
    return uniq


def port_is_free_on(host: str, port: int) -> bool:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        try:
            s.bind((host, port))
        except OSError:
            return False
    return True


def port_is_free(port: int) -> bool:
    return port_is_free_on("0.0.0.0", port)


# =============================================================
# PASSWORT-HASHING
# =============================================================
def hash_password(password: str, salt: str | None = None) -> str:
    if salt is None:
        salt = secrets.token_hex(16)
    h = hashlib.sha256((salt + password).encode("utf-8")).hexdigest()
    return f"sha256${salt}${h}"


def verify_password(password: str, stored: str) -> bool:
    if not stored or not stored.startswith("sha256$"):
        return False
    try:
        _, salt, expected = stored.split("$", 2)
    except ValueError:
        return False
    h = hashlib.sha256((salt + password).encode("utf-8")).hexdigest()
    return secrets.compare_digest(h, expected)


# =============================================================
# KONFIG
# =============================================================
DEFAULT_CONFIG = {
    "bind_ip": "0.0.0.0",
    "port": None,
    "username": "admin",
    "password_hash": None,
    "language": DEFAULT_LANG,
    "allowed_networks": [
        "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
        "127.0.0.0/8", "fc00::/7", "::1/128",
    ],
    "shell_whitelist": None,
    "ssh_targets": [],
    "visible_disks": {},
    "allow_delete": True,
}


def load_config(path: Path) -> dict:
    if path.exists():
        try:
            data = json.loads(path.read_text())
            for k, v in DEFAULT_CONFIG.items():
                data.setdefault(k, v)
            data.pop("mode", None)
            if data.get("language") not in SUPPORTED_LANGS:
                data["language"] = DEFAULT_LANG
            if not port_is_allowed(data.get("port") or 0):
                data["port"] = random_exotic_port()
            if "password" in data and data["password"]:
                data["password_hash"] = hash_password(str(data["password"]))
                del data["password"]
            return data
        except Exception as e:
            print(f"{t('banner.config_unreadable')} {e}")
    cfg = dict(DEFAULT_CONFIG)
    cfg["port"] = random_exotic_port()
    return cfg


def save_config(path: Path, cfg: dict) -> None:
    cfg.pop("mode", None)
    cfg.pop("password", None)
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps(cfg, indent=2, ensure_ascii=False))
    try:
        path.chmod(0o600)
    except Exception:
        pass


# =============================================================
# SSH
# =============================================================
class SSHConnection:
    def __init__(self, name: str, host: str, port: int, user: str,
                 password: str | None = None,
                 key_path: str | None = None,
                 key_pass: str | None = None,
                 sudo_password: str | None = None):
        self.name = name
        self.host = host
        self.port = port
        self.user = user
        self.password = password
        self.key_path = key_path
        self.key_pass = key_pass
        self.sudo_password = sudo_password
        self.client: paramiko.SSHClient | None = None
        self.sftp: paramiko.SFTPClient | None = None
        self.lock = threading.Lock()

    def connect(self, timeout: int = 10) -> None:
        if self.client:
            try:
                t_ = self.client.get_transport()
                if t_ and t_.is_active():
                    return
            except Exception:
                pass
        cli = paramiko.SSHClient()
        cli.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        kwargs = dict(hostname=self.host, port=self.port, username=self.user,
                      timeout=timeout, banner_timeout=timeout, auth_timeout=timeout)
        if self.key_path:
            kwargs["key_filename"] = self.key_path
            if self.key_pass:
                kwargs["passphrase"] = self.key_pass
        if self.password:
            kwargs["password"] = self.password
        cli.connect(**kwargs)
        self.client = cli
        try:
            self.sftp = cli.open_sftp()
        except Exception:
            self.sftp = None

    def ensure_sftp(self) -> paramiko.SFTPClient:
        self.connect()
        if self.sftp is None:
            assert self.client is not None
            self.sftp = self.client.open_sftp()
        return self.sftp

    def close(self) -> None:
        try:
            if self.sftp: self.sftp.close()
        except Exception:
            pass
        try:
            if self.client: self.client.close()
        except Exception:
            pass
        self.sftp = None
        self.client = None

    def _wrap_sudo(self, cmd: str) -> str:
        if not self.sudo_password:
            return cmd
        pw = self.sudo_password.replace("'", "'\\''")
        inner = cmd.replace("'", "'\\''")
        return f"printf '%s\\n' '{pw}' | sudo -S -p '' bash -c '{inner}'"

    def run(self, cmd, timeout: int = 30, sudo: bool = False) -> dict:
        with self.lock:
            self.connect()
            assert self.client is not None
            if isinstance(cmd, list):
                cmd = " ".join(shlex.quote(str(c)) for c in cmd)
            if sudo and self.sudo_password:
                cmd = self._wrap_sudo(cmd)
            stdin, stdout, stderr = self.client.exec_command(cmd, timeout=timeout)
            try:
                out = stdout.read().decode("utf-8", errors="replace")
                err = stderr.read().decode("utf-8", errors="replace")
                rc = stdout.channel.recv_exit_status()
            except socket.timeout:
                return {"returncode": -1, "stdout": "",
                        "stderr": t("err.timeout", s=timeout)}
            return {"returncode": rc, "stdout": out, "stderr": err}


class SSHManager:
    def __init__(self, targets: list[dict]):
        self._targets = targets
        self._conns: dict[str, SSHConnection] = {}

    def update_targets(self, targets: list[dict]) -> None:
        self._targets = targets
        names = {t_["name"] for t_ in targets}
        for n in list(self._conns.keys()):
            if n not in names:
                self._conns[n].close()
                del self._conns[n]

    def get(self, name: str) -> SSHConnection:
        for t_ in self._targets:
            if t_["name"] == name:
                if name not in self._conns:
                    self._conns[name] = SSHConnection(
                        name=t_["name"], host=t_["host"], port=int(t_.get("port", 22)),
                        user=t_["user"],
                        password=t_.get("password"),
                        key_path=t_.get("key_path"),
                        key_pass=t_.get("key_pass"),
                        sudo_password=t_.get("sudo_password"),
                    )
                return self._conns[name]
        raise HTTPException(404, t("err.no_ssh_target", name=name))

    def close_all(self) -> None:
        for c in self._conns.values():
            c.close()
        self._conns.clear()


# =============================================================
# ERSTER START
# =============================================================
def interactive_setup(config_path: Path) -> tuple[dict, Path]:
    print()
    print("=" * 62)
    print(f"  {t('setup.title')}")
    print("=" * 62)
    print()
    print(f"  {t('setup.config_path')} {config_path}")
    print()

    default_lan = get_lan_ip()
    print(t("setup.available_ips"))
    for item in list_all_ips():
        mark = f"  {t('setup.recommended')}" if item["ip"] == default_lan else ""
        print(f"  - {item['ip']:<40} {item['iface']} {item['family']}{mark}")
    print()
    bind_ip = input(f"{t('setup.bind_ip')} [0.0.0.0]: ").strip() or "0.0.0.0"
    try:
        ipaddress.ip_address(bind_ip)
    except ValueError:
        print(f"  ! {t('setup.invalid_ip', ip=bind_ip)}")
        bind_ip = "0.0.0.0"

    suggested = random_exotic_port()
    while True:
        raw = input(f"{t('setup.port')} [{suggested}]: ").strip()
        if not raw:
            port = suggested
            break
        try:
            port = int(raw)
        except ValueError:
            print(f"  ! {t('setup.not_a_number')}")
            continue
        if not port_is_allowed(port):
            if port in FORBIDDEN_PORTS:
                print(f"  ! {t('setup.reserved_port', port=port)}")
            else:
                print(f"  ! {t('setup.invalid_port')}")
            continue
        if not port_is_free_on(bind_ip, port):
            print(f"  ! {t('setup.port_busy', port=port, host=bind_ip)}")
            continue
        break

    username = input(f"{t('setup.username')} [admin]: ").strip() or "admin"
    while True:
        pw1 = input(f"{t('setup.password')}: ").strip()
        if not pw1:
            print(f"  ! {t('setup.password_empty')}")
            continue
        pw2 = input(f"{t('setup.password_repeat')}: ").strip()
        if pw1 != pw2:
            print(f"  ! {t('setup.password_mismatch')}")
            continue
        break

    cfg = {
        **DEFAULT_CONFIG,
        "bind_ip": bind_ip,
        "port": port,
        "username": username,
        "password_hash": hash_password(pw1),
        "language": _ACTIVE_LANG,
    }
    save_config(config_path, cfg)

    print()
    print("=" * 62)
    print(f"  {t('setup.done_title')}")
    print("=" * 62)
    print(f"  {t('setup.done_config')} {config_path}")
    print(f"  {t('setup.done_bind')} {bind_ip}:{port}")
    print(f"  {t('setup.done_user')} {username}")
    print("=" * 62)
    print()

    return cfg, config_path


# =============================================================
# HELFER
# =============================================================
def local_run_cmd(cmd, timeout: int = 30) -> dict:
    try:
        if isinstance(cmd, str):
            r = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=timeout)
        else:
            r = subprocess.run(cmd, capture_output=True, text=True, timeout=timeout)
        return {"returncode": r.returncode, "stdout": r.stdout, "stderr": r.stderr}
    except subprocess.TimeoutExpired:
        return {"returncode": -1, "stdout": "", "stderr": t("err.timeout", s=timeout)}
    except FileNotFoundError as e:
        return {"returncode": -1, "stdout": "", "stderr": str(e)}


def valid_unit(name: str) -> bool:
    if not name or len(name) > 200:
        return False
    ok = set("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@._-")
    return all(c in ok for c in name)


# =============================================================
# APP
# =============================================================
def build_app(CONFIG: dict, config_path: Path) -> FastAPI:
    security = HTTPBasic()
    ssh = SSHManager(CONFIG.get("ssh_targets", []))

    def lang() -> str:
        return CONFIG.get("language") or DEFAULT_LANG

    def check_auth(creds: HTTPBasicCredentials = Depends(security)) -> str:
        ok_user = secrets.compare_digest(creds.username, CONFIG["username"])
        stored = CONFIG.get("password_hash") or ""
        ok_pass = verify_password(creds.password, stored)
        if not (ok_user and ok_pass):
            raise HTTPException(
                status_code=401,
                detail=t("err.auth", lang=lang()),
                headers={"WWW-Authenticate": "Basic"},
            )
        return creds.username

    allowed_nets = [ipaddress.ip_network(n) for n in CONFIG["allowed_networks"]]

    def ip_allowed(ip: str) -> bool:
        try:
            addr = ipaddress.ip_address(ip)
        except ValueError:
            return False
        return any(addr in net for net in allowed_nets)

    app = FastAPI(title=APP_NAME, docs_url=None, redoc_url=None)

    @app.middleware("http")
    async def network_guard(request: Request, call_next):
        client_ip = request.client.host if request.client else ""
        if not ip_allowed(client_ip):
            return JSONResponse(
                status_code=403,
                content={"detail": t("err.network", lang=lang(), ip=client_ip)},
            )
        return await call_next(request)

    def runner_for(overlay: str):
        if overlay == "local":
            return local_run_cmd
        conn = ssh.get(overlay)
        def _run(cmd, timeout: int = 30):
            return conn.run(cmd, timeout=timeout)
        return _run

    def ensure_overlay(overlay: str) -> str:
        if overlay == "local":
            return overlay
        names = [t_["name"] for t_ in CONFIG.get("ssh_targets", [])]
        if overlay not in names:
            raise HTTPException(404, t("err.overlay_not_found", lang=lang(), name=overlay))
        return overlay

    def visible_disks_for(overlay: str) -> list[str] | None:
        mapping = CONFIG.get("visible_disks", {}) or {}
        if overlay in mapping:
            return list(mapping[overlay])
        return None

    def filter_disks(overlay: str, disks: list[dict]) -> list[dict]:
        vis = visible_disks_for(overlay)
        if vis is None:
            return [d for d in disks if d.get("total", 0) >= DISK_MIN_AUTO_BYTES]
        vis_set = set(vis)
        return [d for d in disks if d["mountpoint"] in vis_set]

    def ssh_has_sudo(overlay: str) -> bool:
        if overlay == "local":
            return False
        try:
            conn = ssh.get(overlay)
        except HTTPException:
            return False
        return bool(conn.sudo_password)

    # ---------- Overlays ----------
    @app.get("/api/overlays")
    def api_overlays(_=Depends(check_auth)):
        targets = []
        for t_ in CONFIG.get("ssh_targets", []):
            targets.append({
                "name": t_["name"], "host": t_["host"],
                "port": int(t_.get("port", 22)), "user": t_["user"],
                "auth": t_.get("auth", "password"),
                "key_path": t_.get("key_path"),
                "has_sudo": bool(t_.get("sudo_password")),
            })
        return {
            "app": APP_NAME, "hostname": socket.gethostname(),
            "bind_ip": CONFIG["bind_ip"], "port": CONFIG["port"],
            "language": lang(),
            "available_languages": list(SUPPORTED_LANGS),
            "disk_min_auto_bytes": DISK_MIN_AUTO_BYTES,
            "overlays": [{"name": "local", "label": t("ui.overlay_local", lang=lang()),
                          "type": "local"}] +
                        [{"name": t_["name"], "label": f"SSH: {t_['name']}",
                          "type": "ssh", **t_} for t_ in targets],
        }

    # ---------- System ----------
    @app.get("/api/system")
    def api_system(overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)

        if overlay == "local":
            vm = psutil.virtual_memory()
            boot = datetime.fromtimestamp(psutil.boot_time())
            try:
                load = os.getloadavg()
            except (OSError, AttributeError):
                load = (0.0, 0.0, 0.0)
            all_disks = []
            seen = set()
            for part in psutil.disk_partitions(all=False):
                if part.mountpoint in seen:
                    continue
                seen.add(part.mountpoint)
                try:
                    usage = psutil.disk_usage(part.mountpoint)
                except (PermissionError, OSError):
                    continue
                if usage.total == 0:
                    continue
                all_disks.append({
                    "device": part.device, "mountpoint": part.mountpoint,
                    "fstype": part.fstype, "opts": part.opts,
                    "total": usage.total, "used": usage.used,
                    "free": usage.free, "percent": usage.percent,
                    "small": usage.total < DISK_MIN_AUTO_BYTES,
                })
            if not all_disks:
                try:
                    du = psutil.disk_usage("/")
                    all_disks.append({
                        "device": "?", "mountpoint": "/", "fstype": "?", "opts": "",
                        "total": du.total, "used": du.used, "free": du.free,
                        "percent": du.percent,
                        "small": du.total < DISK_MIN_AUTO_BYTES,
                    })
                except Exception:
                    pass
            all_disks.sort(key=lambda d: (d["mountpoint"] != "/", d["mountpoint"]))
            shown = filter_disks(overlay, all_disks)
            root = next((d for d in shown if d["mountpoint"] == "/"),
                        shown[0] if shown else None)
            return {
                "overlay": "local",
                "hostname": socket.gethostname(),
                "platform": f"{platform.system()} {platform.release()}",
                "python": platform.python_version(),
                "uptime_sec": int((datetime.now() - boot).total_seconds()),
                "cpu_percent": psutil.cpu_percent(interval=0.2),
                "cpu_count": psutil.cpu_count(),
                "load_avg": load,
                "mem_total": vm.total, "mem_used": vm.used, "mem_percent": vm.percent,
                "disk_total": root["total"] if root else 0,
                "disk_used":  root["used"]  if root else 0,
                "disk_percent": root["percent"] if root else 0.0,
                "disks": shown, "all_disks": all_disks,
                "visible_disks": visible_disks_for(overlay),
                "auto_filter_bytes": DISK_MIN_AUTO_BYTES,
                "has_sudo": False,
                "bind_ip": CONFIG["bind_ip"], "port": CONFIG["port"],
            }

        conn = ssh.get(overlay)
        script = (
            "echo HOSTNAME=$(hostname);"
            "echo UNAME=$(uname -s -r);"
            "echo UPTIME=$(awk '{print int($1)}' /proc/uptime);"
            "echo CPUS=$(nproc);"
            "echo LOAD=$(awk '{print $1\" \"$2\" \"$3}' /proc/loadavg);"
            "free -b | awk '/Mem:/ {print \"MEM=\"$2\" \"$3}';"
            "df -B1 -x tmpfs -x devtmpfs -x squashfs -x overlay "
            "--output=source,fstype,size,used,avail,pcent,target 2>/dev/null "
            "| awk 'NR>1 {print \"DISK=\"$1\"|\"$2\"|\"$3\"|\"$4\"|\"$5\"|\"$6\"|\"$7}';"
            "awk '/cpu / {idle=$5; total=$2+$3+$4+$5+$6+$7+$8; print \"CPU=\"(total-idle)\"/\"total}' /proc/stat"
        )
        r = conn.run(script, timeout=12)
        vals: dict[str, str] = {}
        disks_raw: list[str] = []
        for line in r["stdout"].splitlines():
            if line.startswith("DISK="):
                disks_raw.append(line[5:])
                continue
            if "=" in line:
                k, v = line.split("=", 1)
                vals[k.strip()] = v.strip()

        def _int(x, d=0):
            try:
                return int(x)
            except Exception:
                return d

        mem_total = mem_used = 0
        if "MEM" in vals:
            parts = vals["MEM"].split()
            if len(parts) >= 2:
                mem_total, mem_used = _int(parts[0]), _int(parts[1])

        all_disks = []
        for raw in disks_raw:
            cols = raw.split("|")
            if len(cols) < 7:
                continue
            device, fstype, size, used, avail, pcent, mount = cols[:7]
            try:
                total = int(size); used_i = int(used); free_i = int(avail)
            except ValueError:
                continue
            try:
                percent = float(pcent.rstrip("%"))
            except ValueError:
                percent = (100.0 * used_i / total) if total else 0.0
            all_disks.append({
                "device": device, "mountpoint": mount, "fstype": fstype, "opts": "",
                "total": total, "used": used_i, "free": free_i, "percent": percent,
                "small": total < DISK_MIN_AUTO_BYTES,
            })
        if not all_disks:
            r2 = conn.run("df -B1 / | awk 'NR==2 {print $2\" \"$3\" \"$4\" \"$5}'", timeout=8)
            parts = r2["stdout"].split()
            if len(parts) >= 4:
                try:
                    total_i = int(parts[0])
                    all_disks.append({
                        "device": "/", "mountpoint": "/", "fstype": "?", "opts": "",
                        "total": total_i, "used": int(parts[1]),
                        "free": int(parts[2]),
                        "percent": float(parts[3].rstrip("%")),
                        "small": total_i < DISK_MIN_AUTO_BYTES,
                    })
                except ValueError:
                    pass
        all_disks.sort(key=lambda d: (d["mountpoint"] != "/", d["mountpoint"]))
        shown = filter_disks(overlay, all_disks)
        root = next((d for d in shown if d["mountpoint"] == "/"),
                    shown[0] if shown else None)

        load_avg = (0.0, 0.0, 0.0)
        if "LOAD" in vals:
            try:
                load_avg = tuple(float(x) for x in vals["LOAD"].split()[:3])
            except Exception:
                pass
        cpu_percent = 0.0
        if "CPU" in vals and "/" in vals["CPU"]:
            try:
                used, total = vals["CPU"].split("/")
                cpu_percent = 100.0 * float(used) / float(total)
            except Exception:
                pass

        return {
            "overlay": overlay,
            "hostname": vals.get("HOSTNAME", overlay),
            "platform": vals.get("UNAME", "?"),
            "python": "–",
            "uptime_sec": _int(vals.get("UPTIME"), 0),
            "cpu_percent": cpu_percent,
            "cpu_count": _int(vals.get("CPUS"), 1),
            "load_avg": load_avg,
            "mem_total": mem_total, "mem_used": mem_used,
            "mem_percent": (100.0 * mem_used / mem_total) if mem_total else 0.0,
            "disk_total": root["total"] if root else 0,
            "disk_used":  root["used"]  if root else 0,
            "disk_percent": root["percent"] if root else 0.0,
            "disks": shown, "all_disks": all_disks,
            "visible_disks": visible_disks_for(overlay),
            "auto_filter_bytes": DISK_MIN_AUTO_BYTES,
            "has_sudo": ssh_has_sudo(overlay),
            "bind_ip": CONFIG["bind_ip"], "port": CONFIG["port"],
        }

    # ---------- Disks ----------
    @app.get("/api/disks/visibility")
    def api_disk_visibility_get(overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        return {"overlay": overlay, "visible": visible_disks_for(overlay)}

    @app.post("/api/disks/visibility")
    async def api_disk_visibility_set(request: Request, overlay: str = "local",
                                      _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        body = await request.json()
        mode_sel = str(body.get("mode", "auto"))
        mounts = body.get("mounts", [])
        if not isinstance(mounts, list):
            raise HTTPException(400, "mounts must be a list")
        mounts = [str(m) for m in mounts]
        mapping = CONFIG.setdefault("visible_disks", {})
        if mode_sel == "auto":
            mapping.pop(overlay, None)
        else:
            mapping[overlay] = mounts
        save_config(config_path, CONFIG)
        return {"ok": True, "visible": visible_disks_for(overlay)}

    # ---------- Prozesse ----------
    @app.get("/api/processes")
    def api_processes(overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        if overlay == "local":
            procs = []
            for p in psutil.process_iter(
                ["pid", "name", "username", "cpu_percent", "memory_percent", "cmdline"]
            ):
                try:
                    info = p.info
                    info["cmdline"] = " ".join(info.get("cmdline") or [])[:200]
                    procs.append(info)
                except (psutil.NoSuchProcess, psutil.AccessDenied):
                    continue
            procs.sort(key=lambda x: x.get("cpu_percent") or 0, reverse=True)
            return procs[:150]

        cmd = "ps -eo pid,user:20,%cpu,%mem,comm,args --sort=-%cpu --no-headers | head -n 150"
        r = ssh.get(overlay).run(cmd, timeout=15)
        procs = []
        for line in r["stdout"].splitlines():
            parts = line.split(None, 5)
            if len(parts) < 5:
                continue
            try:
                pid = int(parts[0]); user = parts[1]
                cpu = float(parts[2]); mem = float(parts[3])
                name = parts[4]
                cmdline = parts[5] if len(parts) > 5 else name
            except ValueError:
                continue
            procs.append({
                "pid": pid, "name": name, "username": user,
                "cpu_percent": cpu, "memory_percent": mem, "cmdline": cmdline,
            })
        return procs

    @app.post("/api/processes/{pid}/kill")
    def api_kill(pid: int, overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        if overlay == "local":
            try:
                psutil.Process(pid).terminate()
                return {"ok": True}
            except Exception as e:
                raise HTTPException(400, str(e))
        conn = ssh.get(overlay)
        r = conn.run(f"kill {pid}", timeout=10)
        if r["returncode"] != 0 and conn.sudo_password:
            r = conn.run(f"kill {pid}", timeout=10, sudo=True)
        if r["returncode"] != 0:
            raise HTTPException(400, r["stderr"] or "kill failed")
        return {"ok": True}

    # ---------- Dienste ----------
    @app.get("/api/services")
    def api_services(overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        run = runner_for(overlay)
        r = run(["systemctl", "list-units", "--type=service", "--all",
                 "--no-pager", "--no-legend", "--plain"])
        services = []
        for line in r["stdout"].splitlines():
            parts = line.split(None, 4)
            if len(parts) >= 4:
                services.append({"name": parts[0], "load": parts[1],
                                 "active": parts[2], "sub": parts[3]})
        return services

    @app.post("/api/services/{name}/{action}")
    def api_service_action(name: str, action: str, overlay: str = "local",
                           _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        if action not in {"start", "stop", "restart", "reload"}:
            raise HTTPException(400, t("err.invalid_action", lang=lang()))
        if not valid_unit(name):
            raise HTTPException(400, t("err.invalid_service", lang=lang()))
        if overlay == "local":
            return local_run_cmd(["systemctl", action, name])
        conn = ssh.get(overlay)
        r = conn.run(["systemctl", action, name])
        if r["returncode"] != 0 and conn.sudo_password:
            r = conn.run(["systemctl", action, name], sudo=True)
        return r

    @app.get("/api/services/{name}/logs")
    def api_service_logs(name: str, lines: int = 200, overlay: str = "local",
                         _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        if not valid_unit(name):
            raise HTTPException(400, t("err.invalid_service", lang=lang()))
        lines = max(1, min(lines, 5000))
        run = runner_for(overlay)
        r = run(["journalctl", "-u", name, "-n", str(lines), "--no-pager"])
        return {"output": r["stdout"] or r["stderr"]}

    # ---------- Shell ----------
    @app.post("/api/shell")
    async def api_shell(request: Request, overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        body = await request.json()
        cmd = (body.get("cmd") or "").strip()
        use_sudo = bool(body.get("sudo"))
        if not cmd:
            raise HTTPException(400, t("err.no_cmd", lang=lang()))
        wl = CONFIG.get("shell_whitelist")
        if wl is not None and not any(cmd.startswith(p) for p in wl):
            raise HTTPException(403, t("err.cmd_not_allowed", lang=lang()))
        if overlay == "local":
            return local_run_cmd(cmd, timeout=60)
        conn = ssh.get(overlay)
        return conn.run(cmd, timeout=60, sudo=use_sudo)

    # ---------- Dateien ----------
    @app.get("/api/files")
    def api_files(path: str = "/", overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        if overlay == "local":
            p = Path(path).expanduser().resolve()
            if not p.exists():
                raise HTTPException(404, t("err.path_not_found", lang=lang()))
            if p.is_file():
                try:
                    content = p.read_text(errors="replace")[:200_000]
                except Exception as e:
                    raise HTTPException(400, str(e))
                st = p.stat()
                return {"type": "file", "path": str(p), "content": content,
                        "size": st.st_size, "mode": oct(st.st_mode & 0o777),
                        "uid": st.st_uid, "gid": st.st_gid}
            entries = []
            try:
                items = sorted(p.iterdir(), key=lambda x: (not x.is_dir(), x.name.lower()))
            except PermissionError:
                raise HTTPException(403, t("err.no_access", lang=lang()))
            for c in items:
                try:
                    st = c.stat()
                    entries.append({"name": c.name, "is_dir": c.is_dir(),
                                    "size": st.st_size, "mtime": int(st.st_mtime),
                                    "mode": oct(st.st_mode & 0o777),
                                    "uid": st.st_uid, "gid": st.st_gid})
                except (PermissionError, OSError):
                    continue
            return {"type": "dir", "path": str(p), "entries": entries}

        conn = ssh.get(overlay)
        cmd = f"ls -la --time-style=+%s -- {shlex.quote(path)}"
        r = conn.run(cmd, timeout=10)
        if r["returncode"] != 0 and conn.sudo_password:
            r = conn.run(cmd, timeout=10, sudo=True)
        if r["returncode"] != 0:
            raise HTTPException(400, r["stderr"] or "ls failed")
        entries = []
        for line in r["stdout"].splitlines()[1:]:
            parts = line.split(None, 6)
            if len(parts) < 7:
                continue
            perms, _, owner, group, size_s, mtime, name = parts
            if name in (".", ".."):
                continue
            is_dir = perms.startswith("d")
            size = 0
            if not is_dir:
                try:
                    size = int(size_s)
                except Exception:
                    size = 0
            try:
                mtime = int(mtime)
            except Exception:
                mtime = 0
            entries.append({"name": name, "is_dir": is_dir, "size": size,
                            "mtime": mtime, "mode": perms,
                            "owner": owner, "group": group})
        entries.sort(key=lambda x: (not x["is_dir"], x["name"].lower()))
        abs_path = path.rstrip("/") or "/"
        return {"type": "dir", "path": abs_path, "entries": entries}

    # ---------- Editor ----------
    @app.get("/api/files/edit")
    def api_file_edit(path: str, overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        if overlay == "local":
            p = Path(path).expanduser().resolve()
            if not p.is_file():
                raise HTTPException(404, t("err.no_file", lang=lang()))
            try:
                content = p.read_text(errors="replace")
            except Exception as e:
                raise HTTPException(400, str(e))
            if len(content) > 2_000_000:
                raise HTTPException(413, t("err.file_too_big", lang=lang()))
            return {"path": str(p), "content": content, "size": p.stat().st_size}
        conn = ssh.get(overlay)
        r = conn.run(f"cat -- {shlex.quote(path)}", timeout=30)
        if r["returncode"] != 0:
            raise HTTPException(400, r["stderr"] or "read failed")
        if len(r["stdout"]) > 2_000_000:
            raise HTTPException(413, t("err.file_too_big", lang=lang()))
        return {"path": path, "content": r["stdout"], "size": len(r["stdout"])}

    @app.post("/api/files/save")
    async def api_file_save(request: Request, overlay: str = "local",
                            _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        body = await request.json()
        path = str(body.get("path", "")).strip()
        content = body.get("content", "")
        if not path:
            raise HTTPException(400, t("err.path_missing", lang=lang()))
        if overlay == "local":
            p = Path(path).expanduser().resolve()
            try:
                p.write_text(content)
            except PermissionError:
                raise HTTPException(403, t("err.no_write", lang=lang()))
            return {"ok": True}
        conn = ssh.get(overlay)
        tmp_name = f".pcoverlay-edit-{uuid.uuid4().hex}.tmp"
        tmp_path = f"/tmp/{tmp_name}"
        sftp = conn.ensure_sftp()
        try:
            with sftp.open(tmp_path, "w") as f:
                f.write(content)
        except Exception as e:
            raise HTTPException(400, t("err.tmp_write", lang=lang(), err=e))
        mv = f"mv -f -- {shlex.quote(tmp_path)} {shlex.quote(path)}"
        r = conn.run(mv, timeout=30)
        if r["returncode"] != 0 and conn.sudo_password:
            r = conn.run(mv, timeout=30, sudo=True)
        if r["returncode"] != 0:
            try:
                conn.run(f"rm -f -- {shlex.quote(tmp_path)}", timeout=10)
            except Exception:
                pass
            raise HTTPException(400, r["stderr"] or t("err.save_failed", lang=lang()))
        return {"ok": True}

    # ---------- Download ----------
    @app.get("/api/files/download")
    def api_download(path: str, overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        if overlay == "local":
            p = Path(path).expanduser().resolve()
            if not p.is_file():
                raise HTTPException(404, t("err.file_not_found", lang=lang()))
            return FileResponse(str(p), filename=p.name)
        conn = ssh.get(overlay)
        sftp = conn.ensure_sftp()
        try:
            buf = io.BytesIO()
            sftp.getfo(path, buf)
            buf.seek(0)
            data = buf.read()
            fname = path.rsplit("/", 1)[-1] or "download"
            return HTMLResponse(data, media_type="application/octet-stream",
                                headers={"Content-Disposition":
                                         f'attachment; filename="{fname}"'})
        except Exception as e:
            raise HTTPException(400, str(e))

    @app.post("/api/files/download_zip")
    async def api_download_zip(request: Request, overlay: str = "local",
                               _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        body = await request.json()
        paths = body.get("paths", [])
        if not isinstance(paths, list) or not paths:
            raise HTTPException(400, t("err.no_paths", lang=lang()))
        paths = [str(p) for p in paths]
        buf = io.BytesIO()
        try:
            if overlay == "local":
                _zip_local(paths, buf)
            else:
                conn = ssh.get(overlay)
                sftp = conn.ensure_sftp()
                _zip_remote(sftp, paths, buf, conn)
        except HTTPException:
            raise
        except Exception as e:
            raise HTTPException(400, t("err.zip_failed", lang=lang(), err=e))
        buf.seek(0)
        data = buf.read()
        fname = f"pcoverlay-{int(time.time())}.zip"
        return HTMLResponse(data, media_type="application/zip",
                            headers={"Content-Disposition":
                                     f'attachment; filename="{fname}"'})

    def _zip_local(paths: list[str], buf: io.BytesIO) -> None:
        with zipfile.ZipFile(buf, "w", zipfile.ZIP_DEFLATED) as zf:
            for p in paths:
                src = Path(p).expanduser().resolve()
                if not src.exists():
                    continue
                if src.is_file():
                    zf.write(src, arcname=src.name)
                else:
                    base_parent = src.parent
                    for root, dirs, files in os.walk(src):
                        for fn in files:
                            full = Path(root) / fn
                            arc = full.relative_to(base_parent)
                            try:
                                zf.write(full, arcname=str(arc))
                            except Exception:
                                continue

    def _zip_remote(sftp: paramiko.SFTPClient, paths: list[str],
                    buf: io.BytesIO, conn: SSHConnection | None = None) -> None:
        with zipfile.ZipFile(buf, "w", zipfile.ZIP_DEFLATED) as zf:
            for p in paths:
                _zip_remote_add(sftp, p, zf, conn=conn)

    def _zip_remote_add(sftp: paramiko.SFTPClient, path: str,
                        zf: zipfile.ZipFile, base: str | None = None,
                        conn: SSHConnection | None = None) -> None:
        try:
            st = sftp.stat(path)
        except Exception:
            return
        name = path.rstrip("/").rsplit("/", 1)[-1] or "item"
        arc = name if base is None else base.rstrip("/") + "/" + name
        import stat as statmod
        if statmod.S_ISDIR(st.st_mode):
            try:
                for entry in sftp.listdir_attr(path):
                    child = path.rstrip("/") + "/" + entry.filename
                    _zip_remote_add(sftp, child, zf, base=arc, conn=conn)
            except Exception:
                return
        else:
            try:
                with sftp.open(path, "rb") as f:
                    data = f.read()
                zf.writestr(arc, data)
            except Exception:
                if conn and conn.sudo_password:
                    r = conn.run(f"cat -- {shlex.quote(path)} | base64", timeout=120, sudo=True)
                    if r["returncode"] == 0:
                        import base64 as _b64
                        try:
                            data = _b64.b64decode(r["stdout"])
                            zf.writestr(arc, data)
                        except Exception:
                            return

    # ---------- Upload ----------
    @app.post("/api/files/upload")
    async def api_upload(
        file: UploadFile = File(...),
        path: str = Form(...),
        overlay: str = Form("local"),
        _=Depends(check_auth),
    ):
        overlay = ensure_overlay(overlay)
        target_dir = path
        if not target_dir.endswith("/"):
            target_dir += "/"
        filename = file.filename or "upload.bin"
        filename = filename.replace("\\", "/").split("/")[-1] or "upload.bin"

        if overlay == "local":
            base = Path(target_dir).expanduser().resolve()
            if not base.is_dir():
                raise HTTPException(400, t("err.target_dir_invalid", lang=lang()))
            dest = base / filename
            try:
                with open(dest, "wb") as f:
                    while True:
                        chunk = await file.read(1024 * 1024)
                        if not chunk:
                            break
                        f.write(chunk)
            except PermissionError:
                raise HTTPException(403, t("err.no_write_dir", lang=lang()))
            except Exception as e:
                raise HTTPException(400, t("err.write_failed", lang=lang(), err=e))
            return {"ok": True, "path": str(dest), "size": dest.stat().st_size, "mode": "direct"}

        conn = ssh.get(overlay)
        remote_path = target_dir + filename
        sftp = conn.ensure_sftp()

        direct_error: Exception | None = None
        try:
            with sftp.open(remote_path, "wb") as f:
                f.set_pipelined(True)
                while True:
                    chunk = await file.read(1024 * 1024)
                    if not chunk:
                        break
                    f.write(chunk)
            try:
                size = sftp.stat(remote_path).st_size
            except Exception:
                size = 0
            return {"ok": True, "path": remote_path, "size": size, "mode": "sftp"}
        except Exception as e:
            direct_error = e
            if not conn.sudo_password:
                raise HTTPException(403, t("err.no_write_sudo", lang=lang(), path=remote_path))

        try:
            await file.seek(0)
        except Exception:
            pass

        tmp_name = f".pcoverlay-upload-{uuid.uuid4().hex}.bin"
        tmp_path = f"/tmp/{tmp_name}"
        try:
            with sftp.open(tmp_path, "wb") as f:
                f.set_pipelined(True)
                while True:
                    chunk = await file.read(1024 * 1024)
                    if not chunk:
                        break
                    f.write(chunk)
        except Exception as e:
            raise HTTPException(400, t("err.tmp_write", lang=lang(), err=e))

        try:
            conn.run(f"mkdir -p -- {shlex.quote(target_dir)}", timeout=30, sudo=True)
            mv = (f"mv -f -- {shlex.quote(tmp_path)} {shlex.quote(remote_path)} "
                  f"&& chmod 0644 -- {shlex.quote(remote_path)}")
            r = conn.run(mv, timeout=60, sudo=True)
            if r["returncode"] != 0:
                conn.run(f"rm -f -- {shlex.quote(tmp_path)}", timeout=15, sudo=True)
                raise HTTPException(400, t("err.mv_failed_sudo", lang=lang(), err=r['stderr'].strip()))
        except HTTPException:
            raise
        except Exception as e:
            try:
                conn.run(f"rm -f -- {shlex.quote(tmp_path)}", timeout=15, sudo=True)
            except Exception:
                pass
            raise HTTPException(400, t("err.sudo_upload", lang=lang(), err=e))

        try:
            size = sftp.stat(remote_path).st_size
        except Exception:
            size = 0
        return {"ok": True, "path": remote_path, "size": size, "mode": "sudo"}

    # ---------- Delete/Rename/mkdir/chmod/chown ----------
    @app.post("/api/files/delete")
    async def api_delete(request: Request, overlay: str = "local", _=Depends(check_auth)):
        if not CONFIG.get("allow_delete", True):
            raise HTTPException(403, t("err.delete_disabled", lang=lang()))
        overlay = ensure_overlay(overlay)
        body = await request.json()
        paths = body.get("paths", [])
        if not isinstance(paths, list) or not paths:
            raise HTTPException(400, t("err.no_paths", lang=lang()))
        paths = [str(p) for p in paths]
        results = []

        if overlay == "local":
            for p in paths:
                try:
                    target = Path(p).expanduser().resolve()
                    if not target.exists():
                        results.append({"path": p, "ok": False, "error": "does not exist"})
                        continue
                    if target.is_dir():
                        shutil.rmtree(target)
                    else:
                        target.unlink()
                    results.append({"path": p, "ok": True})
                except Exception as e:
                    results.append({"path": p, "ok": False, "error": str(e)})
        else:
            conn = ssh.get(overlay)
            for p in paths:
                q = shlex.quote(p)
                r = conn.run(f"rm -rf -- {q}", timeout=60)
                if r["returncode"] != 0 and conn.sudo_password:
                    r2 = conn.run(f"rm -rf -- {q}", timeout=60, sudo=True)
                    if r2["returncode"] == 0:
                        results.append({"path": p, "ok": True, "mode": "sudo"})
                        continue
                    results.append({"path": p, "ok": False, "error": r2["stderr"].strip()})
                    continue
                results.append({"path": p, "ok": r["returncode"] == 0,
                                "error": r["stderr"].strip() if r["returncode"] != 0 else "",
                                "mode": "direct" if r["returncode"] == 0 else "fail"})
        return {"ok": all(x["ok"] for x in results), "results": results}

    @app.post("/api/files/rename")
    async def api_rename(request: Request, overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        body = await request.json()
        path = str(body.get("path", "")).strip()
        new_name = str(body.get("new_name", "")).strip()
        if not path or not new_name:
            raise HTTPException(400, t("err.name_required", lang=lang()))
        if "/" in new_name or "\\" in new_name or new_name in (".", ".."):
            raise HTTPException(400, t("err.invalid_name", lang=lang()))
        parent = path.rstrip("/").rsplit("/", 1)[0] or "/"
        if parent == "":
            parent = "/"
        new_path = parent.rstrip("/") + "/" + new_name

        if overlay == "local":
            src = Path(path).expanduser().resolve()
            dst = Path(new_path).expanduser().resolve()
            if not src.exists():
                raise HTTPException(404, t("err.src_not_found", lang=lang()))
            if dst.exists():
                raise HTTPException(400, t("err.dst_exists", lang=lang()))
            try:
                src.rename(dst)
            except Exception as e:
                raise HTTPException(400, str(e))
            return {"ok": True, "path": str(dst)}

        conn = ssh.get(overlay)
        r = conn.run(f"mv -- {shlex.quote(path)} {shlex.quote(new_path)}", timeout=30)
        if r["returncode"] != 0 and conn.sudo_password:
            r2 = conn.run(f"mv -- {shlex.quote(path)} {shlex.quote(new_path)}",
                          timeout=30, sudo=True)
            if r2["returncode"] == 0:
                return {"ok": True, "path": new_path, "mode": "sudo"}
            raise HTTPException(400, r2["stderr"] or t("err.mv_failed", lang=lang()))
        if r["returncode"] != 0:
            raise HTTPException(400, r["stderr"] or t("err.mv_failed", lang=lang()))
        return {"ok": True, "path": new_path}

    @app.post("/api/files/mkdir")
    async def api_mkdir(request: Request, overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        body = await request.json()
        path = str(body.get("path", "")).strip()
        name = str(body.get("name", "")).strip()
        if not path or not name:
            raise HTTPException(400, t("err.name_required", lang=lang()))
        if "/" in name or "\\" in name or name in (".", ".."):
            raise HTTPException(400, t("err.invalid_name", lang=lang()))
        target = (path.rstrip("/") + "/" + name) if path != "/" else ("/" + name)

        if overlay == "local":
            try:
                Path(target).mkdir(parents=False, exist_ok=False)
            except Exception as e:
                raise HTTPException(400, str(e))
            return {"ok": True, "path": target}

        conn = ssh.get(overlay)
        r = conn.run(f"mkdir -- {shlex.quote(target)}", timeout=15)
        if r["returncode"] != 0 and conn.sudo_password:
            r2 = conn.run(f"mkdir -- {shlex.quote(target)}", timeout=15, sudo=True)
            if r2["returncode"] == 0:
                return {"ok": True, "path": target, "mode": "sudo"}
            raise HTTPException(400, r2["stderr"] or t("err.mkdir_failed", lang=lang()))
        if r["returncode"] != 0:
            raise HTTPException(400, r["stderr"] or t("err.mkdir_failed", lang=lang()))
        return {"ok": True, "path": target}

    @app.post("/api/files/chmod")
    async def api_chmod(request: Request, overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        body = await request.json()
        path = str(body.get("path", "")).strip()
        mode_s = str(body.get("mode", "")).strip()
        if not path or not mode_s:
            raise HTTPException(400, t("err.name_required", lang=lang()))
        if not all(c in "01234567" for c in mode_s):
            raise HTTPException(400, t("err.mode_octal", lang=lang()))
        if overlay == "local":
            r = local_run_cmd(["chmod", mode_s, path])
        else:
            conn = ssh.get(overlay)
            r = conn.run(["chmod", mode_s, path])
            if r["returncode"] != 0 and conn.sudo_password:
                r = conn.run(["chmod", mode_s, path], sudo=True)
        if r["returncode"] != 0:
            raise HTTPException(400, r["stderr"] or t("err.chmod_failed", lang=lang()))
        return {"ok": True}

    @app.post("/api/files/chown")
    async def api_chown(request: Request, overlay: str = "local", _=Depends(check_auth)):
        overlay = ensure_overlay(overlay)
        body = await request.json()
        path = str(body.get("path", "")).strip()
        owner = str(body.get("owner", "")).strip()
        if not path or not owner:
            raise HTTPException(400, t("err.name_required", lang=lang()))
        if not all(c.isalnum() or c in ":-._" for c in owner):
            raise HTTPException(400, t("err.invalid_owner", lang=lang()))
        if overlay == "local":
            r = local_run_cmd(["chown", owner, path])
        else:
            conn = ssh.get(overlay)
            r = conn.run(["chown", owner, path])
            if r["returncode"] != 0 and conn.sudo_password:
                r = conn.run(["chown", owner, path], sudo=True)
        if r["returncode"] != 0:
            raise HTTPException(400, r["stderr"] or t("err.chown_failed", lang=lang()))
        return {"ok": True}

    # ---------- SSH-Ziele ----------
    @app.get("/api/ssh_targets")
    def api_ssh_targets(_=Depends(check_auth)):
        out = []
        for t_ in CONFIG.get("ssh_targets", []):
            out.append({
                "name": t_["name"], "host": t_["host"], "port": int(t_.get("port", 22)),
                "user": t_["user"], "auth": t_.get("auth", "password"),
                "key_path": t_.get("key_path"),
                "password": t_.get("password", ""),
                "key_pass": t_.get("key_pass", ""),
                "sudo_password": t_.get("sudo_password", ""),
            })
        return out

    @app.post("/api/ssh_targets")
    async def api_ssh_targets_add(request: Request, _=Depends(check_auth)):
        body = await request.json()
        name = str(body.get("name", "")).strip()
        host = str(body.get("host", "")).strip()
        user_s = str(body.get("user", "")).strip()
        if not (name and host and user_s):
            raise HTTPException(400, t("err.fields_required", lang=lang()))
        if name == "local":
            raise HTTPException(400, t("err.local_reserved", lang=lang()))
        try:
            port = int(body.get("port", 22))
            if not (1 <= port <= 65535):
                raise ValueError
        except (TypeError, ValueError):
            raise HTTPException(400, t("err.invalid_port", lang=lang()))
        auth = str(body.get("auth", "password"))
        if auth not in ("password", "key"):
            raise HTTPException(400, t("err.auth_type", lang=lang()))

        entry = {"name": name, "host": host, "port": port, "user": user_s, "auth": auth}
        if auth == "key":
            entry["key_path"] = str(body.get("key_path", "")).strip()
            kp = str(body.get("key_pass", "") or "")
            if kp:
                entry["key_pass"] = kp
        else:
            pw = str(body.get("password", "") or "")
            if pw:
                entry["password"] = pw

        sp = str(body.get("sudo_password", "") or "")
        if sp:
            entry["sudo_password"] = sp

        existing = CONFIG.setdefault("ssh_targets", [])
        for i, t_ in enumerate(existing):
            if t_["name"] == name:
                for k in ("password", "key_pass", "key_path", "sudo_password"):
                    if k not in entry and k in t_:
                        entry[k] = t_[k]
                if body.get("clear_sudo"):
                    entry.pop("sudo_password", None)
                existing[i] = entry
                break
        else:
            existing.append(entry)
        save_config(config_path, CONFIG)
        ssh.update_targets(existing)
        try:
            conn = ssh.get(name)
            conn.close()
            conn.connect(timeout=8)
            test = {"ok": True, "message": t("ok.conn", lang=lang())}
        except Exception as e:
            test = {"ok": False, "message": t("ok.conn_failed", lang=lang(), err=e)}
        return {"ok": True, "target": entry, "test": test}

    @app.delete("/api/ssh_targets/{name}")
    def api_ssh_targets_delete(name: str, _=Depends(check_auth)):
        existing = CONFIG.get("ssh_targets", [])
        new = [t_ for t_ in existing if t_["name"] != name]
        if len(new) == len(existing):
            raise HTTPException(404, t("err.not_found", lang=lang()))
        CONFIG["ssh_targets"] = new
        CONFIG.get("visible_disks", {}).pop(name, None)
        save_config(config_path, CONFIG)
        ssh.update_targets(new)
        return {"ok": True}

    @app.post("/api/ssh_targets/{name}/test")
    def api_ssh_targets_test(name: str, _=Depends(check_auth)):
        try:
            conn = ssh.get(name)
            conn.close()
            conn.connect(timeout=8)
            return {"ok": True, "message": t("ok.conn", lang=lang())}
        except HTTPException:
            raise
        except Exception as e:
            return {"ok": False, "message": t("ok.test_failed", lang=lang(), err=e)}

    @app.post("/api/ssh_targets/{name}/sudo_password")
    async def api_ssh_targets_sudo(name: str, request: Request, _=Depends(check_auth)):
        body = await request.json()
        pw = str(body.get("password", "") or "")
        if not pw:
            raise HTTPException(400, t("err.password_empty", lang=lang()))
        existing = CONFIG.get("ssh_targets", [])
        found = False
        for t_ in existing:
            if t_["name"] == name:
                t_["sudo_password"] = pw
                found = True
                break
        if not found:
            raise HTTPException(404, t("err.not_found", lang=lang()))
        save_config(config_path, CONFIG)
        ssh.update_targets(existing)
        try:
            conn = ssh.get(name)
            r = conn.run("id", timeout=8, sudo=True)
            if r["returncode"] == 0:
                return {"ok": True, "message": t("ok.sudo_works", lang=lang())}
            return {"ok": False, "message": t("ok.sudo_failed", lang=lang(), err=r['stderr'].strip())}
        except Exception as e:
            return {"ok": False, "message": t("ok.sudo_failed", lang=lang(), err=e)}

    # ---------- Einstellungen ----------
    @app.get("/api/settings")
    def api_settings_get(_=Depends(check_auth)):
        return {
            "app": APP_NAME,
            "bind_ip": CONFIG["bind_ip"], "port": CONFIG["port"],
            "username": CONFIG["username"],
            "language": lang(),
            "available_languages": list(SUPPORTED_LANGS),
            "available_ips": list_all_ips(),
            "allow_delete": CONFIG.get("allow_delete", True),
            "disk_min_auto_bytes": DISK_MIN_AUTO_BYTES,
        }

    @app.post("/api/settings/check_port")
    async def api_check_port(request: Request, _=Depends(check_auth)):
        body = await request.json()
        try:
            port = int(body.get("port"))
        except (TypeError, ValueError):
            return {"ok": False, "reason": t("err.no_valid_port", lang=lang())}
        if not port_is_allowed(port):
            if port in FORBIDDEN_PORTS:
                return {"ok": False, "reason": t("err.port_reserved", lang=lang(), port=port)}
            return {"ok": False, "reason": t("err.port_out_of_range", lang=lang())}
        if not port_is_free(port):
            return {"ok": False, "reason": t("err.port_busy", lang=lang(), port=port)}
        return {"ok": True, "reason": t("ok.port_free", lang=lang(), port=port)}

    @app.post("/api/settings")
    async def api_settings_set(request: Request, _=Depends(check_auth)):
        body = await request.json()
        bind_ip = str(body.get("bind_ip", "")).strip() or "0.0.0.0"
        try:
            ipaddress.ip_address(bind_ip)
        except ValueError:
            raise HTTPException(400, t("err.invalid_bind", lang=lang(), ip=bind_ip))
        try:
            port = int(body.get("port"))
        except (TypeError, ValueError):
            raise HTTPException(400, t("err.invalid_port", lang=lang()))
        if not port_is_allowed(port):
            raise HTTPException(400, t("err.port_not_allowed", lang=lang(), port=port))
        if not port_is_free(port) and port != CONFIG["port"]:
            raise HTTPException(400, t("err.port_busy", lang=lang(), port=port))

        new_user = str(body.get("username", "")).strip() or CONFIG["username"]
        new_pass = str(body.get("password", "")).strip()
        new_lang = str(body.get("language", "")).strip()

        CONFIG["bind_ip"] = bind_ip
        CONFIG["port"] = port
        CONFIG["username"] = new_user
        if new_pass:
            CONFIG["password_hash"] = hash_password(new_pass)
        if new_lang in SUPPORTED_LANGS:
            CONFIG["language"] = new_lang
            global _ACTIVE_LANG
            _ACTIVE_LANG = new_lang
        if "allow_delete" in body:
            CONFIG["allow_delete"] = bool(body["allow_delete"])
        save_config(config_path, CONFIG)

        host = get_lan_ip() if bind_ip in ("0.0.0.0", "::") else bind_ip
        return {
            "ok": True,
            "message": t("ok.saved_restart", lang=lang()),
            "restart_required": True,
            "new_url": f"http://{host}:{port}",
        }

    # ---------- Frontend ----------
    @app.get("/", response_class=HTMLResponse)
    def index(_=Depends(check_auth)):
        return HTMLResponse(render_index(CONFIG.get("language") or DEFAULT_LANG))

    @app.on_event("shutdown")
    def _on_shutdown():
        ssh.close_all()

    return app


def render_index(lang: str) -> str:
    """Rendert das Frontend mit der aktiven Sprache als injiziertes JSON."""
    table = LOCALES.get(lang) or LOCALES[DEFAULT_LANG]
    # Fallback auf Englisch für fehlende Keys
    i18n = {**LOCALES[DEFAULT_LANG], **table}
    # Nur UI-Keys ins Frontend (Setup/Banner/API-Error bleiben im Backend)
    ui = {k: v for k, v in i18n.items() if k.startswith("ui.")}
    lang_json = json.dumps({
        "lang": lang,
        "supported": list(SUPPORTED_LANGS),
        "strings": ui,
    }, ensure_ascii=False)
    html = INDEX_HTML
    html = html.replace("__I18N_JSON__", lang_json)
    html = html.replace("__HTML_LANG__", lang)
    return html


# =============================================================
# HTML-FRONTEND
# =============================================================
# Hinweis: die Sprache wird über window.I18N gesteuert, das im Backend
# beim Rendern der Seite injiziert wird. UI-Strings werden über t("key")
# gezogen. Backend-Fehlertexte kommen bereits übersetzt an.
INDEX_HTML = r"""<!doctype html>
<html lang="__HTML_LANG__">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>PC Overlay</title>
<style>
  :root{--bg:#0f1115;--panel:#171a21;--fg:#e6e6e6;--acc:#4ea1ff;--warn:#ff5f56;--ok:#5fd35f;--muted:#888}
  *{box-sizing:border-box}
  body{margin:0;font-family:system-ui,-apple-system,sans-serif;background:var(--bg);color:var(--fg);font-size:14px}
  header{display:flex;flex-wrap:wrap;gap:12px;align-items:center;padding:10px 20px;background:var(--panel);border-bottom:1px solid #222;position:sticky;top:0;z-index:5}
  header h1{font-size:16px;margin:0 12px 0 0;white-space:nowrap}
  .overlay-switch{display:flex;gap:6px;align-items:center;flex-wrap:wrap;padding:4px 8px;background:#0b0d11;border:1px solid #222;border-radius:8px}
  .overlay-switch label{color:var(--muted);font-size:12px;margin-right:4px}
  .overlay-switch select{background:#0b0d11;border:1px solid #333;color:var(--fg);padding:5px 8px;border-radius:6px;font-size:13px}
  nav{display:flex;gap:6px;flex-wrap:wrap}
  nav button{background:transparent;border:1px solid #333;color:var(--fg);padding:6px 12px;border-radius:6px;cursor:pointer;font-size:13px}
  nav button.active{background:var(--acc);border-color:var(--acc);color:#000}
  main{padding:20px;max-width:1300px;margin:auto}
  .card{background:var(--panel);border:1px solid #222;border-radius:10px;padding:16px;margin-bottom:16px}
  .card h3{margin:0 0 12px;font-size:15px;color:#ccc}
  .grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:12px}
  .stat{background:#0b0d11;padding:12px;border-radius:8px;border:1px solid #1a1d23}
  .stat span{color:var(--muted);font-size:12px;display:block}
  .stat b{display:block;font-size:20px;margin-top:4px;font-weight:600}
  table{width:100%;border-collapse:collapse;font-size:13px}
  th,td{text-align:left;padding:7px 8px;border-bottom:1px solid #1e222a;vertical-align:top}
  th{color:var(--muted);font-weight:500;font-size:12px;text-transform:uppercase;letter-spacing:.03em}
  tr:hover{background:#1b1f28}
  button.act{background:#222;border:1px solid #333;color:var(--fg);padding:4px 9px;border-radius:6px;cursor:pointer;font-size:12px;margin-right:3px;text-decoration:none;display:inline-block}
  button.act:hover{background:#2c313b}
  button.act.danger{background:var(--warn);border-color:var(--warn);color:#000}
  button.act.primary{background:var(--acc);border-color:var(--acc);color:#000}
  input,select,textarea{background:#0b0d11;border:1px solid #333;color:var(--fg);padding:8px 10px;border-radius:6px;font-family:ui-monospace,monospace;font-size:13px}
  input:focus,textarea:focus,select:focus{outline:none;border-color:var(--acc)}
  input[type=checkbox]{width:16px;height:16px;accent-color:var(--acc);cursor:pointer}
  pre{background:#0b0d11;padding:12px;border-radius:6px;overflow:auto;max-height:500px;font-size:12px;line-height:1.45;margin:0;white-space:pre-wrap;word-break:break-all}
  .row{display:flex;gap:8px;align-items:center;flex-wrap:wrap}
  .row input{flex:1;min-width:200px}
  .hidden{display:none}
  .muted{color:var(--muted);font-size:12px}
  .tag{display:inline-block;padding:2px 8px;border-radius:10px;font-size:11px;font-weight:500}
  .tag.running,.tag.active{background:rgba(95,211,95,.15);color:var(--ok)}
  .tag.stopped,.tag.inactive,.tag.failed{background:rgba(255,95,86,.15);color:var(--warn)}
  .tag.ssh{background:rgba(78,161,255,.15);color:var(--acc)}
  .tag.small{background:rgba(255,200,80,.15);color:#ffc850}
  .tag.auto{background:rgba(95,211,95,.15);color:var(--ok)}
  .tag.sudo{background:rgba(168,85,247,.18);color:#c084fc}
  .toast{position:fixed;bottom:20px;right:20px;background:#222;padding:12px 18px;border-radius:8px;border:1px solid #333;opacity:0;transition:opacity .3s;pointer-events:none;z-index:100;max-width:400px}
  .toast.show{opacity:1}
  .spin{display:inline-block;width:12px;height:12px;border:2px solid #444;border-top-color:var(--acc);border-radius:50%;animation:s .8s linear infinite;vertical-align:-2px;margin-right:6px}
  @keyframes s{to{transform:rotate(360deg)}}
  label.field{display:block;color:var(--muted);font-size:12px;margin-bottom:6px}
  .split{display:grid;grid-template-columns:repeat(auto-fit,minmax(260px,1fr));gap:12px}
  .disk-row{display:flex;align-items:center;gap:10px;padding:8px 0;border-bottom:1px solid #1e222a}
  .disk-row:last-child{border-bottom:none}
  .disk-row input[type=checkbox]{width:18px;height:18px}
  .disk-row .disk-label{flex:1}
  .disk-row .disk-bar{flex:0 0 180px;height:8px;background:#0b0d11;border-radius:4px;overflow:hidden;border:1px solid #1a1d23}
  .disk-row .disk-bar > div{height:100%;background:var(--acc)}
  body.disk-edit-active #diskListGrid{opacity:.55;transition:opacity .2s}
  .drop{position:relative;border:2px dashed #333;border-radius:8px;padding:14px;text-align:center;transition:border-color .2s,background .2s;cursor:pointer}
  .drop:hover,.drop.over{border-color:var(--acc);background:rgba(78,161,255,.05)}
  .notice{background:rgba(255,200,80,.08);border:1px solid rgba(255,200,80,.3);color:#ffc850;padding:10px 12px;border-radius:6px;font-size:12px;margin-bottom:12px}
  .notice.ok{background:rgba(95,211,95,.08);border-color:rgba(95,211,95,.3);color:var(--ok)}
  .toolbar{display:flex;gap:8px;flex-wrap:wrap;align-items:center;padding:10px 12px;background:#0b0d11;border:1px solid #1a1d23;border-radius:8px;margin-top:12px}
  .toolbar .sel{color:var(--muted);font-size:12px;margin-right:auto}
  .up-list{display:flex;flex-direction:column;gap:8px;margin-top:10px}
  .up-item{background:#0b0d11;border:1px solid #1a1d23;border-radius:8px;padding:10px}
  .up-item .up-head{display:flex;gap:10px;align-items:center;font-size:12px}
  .up-item .up-name{flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
  .up-item .up-percent{font-variant-numeric:tabular-nums;color:var(--muted)}
  .up-item .up-detail{color:var(--muted);font-size:11px;margin-top:4px}
  .up-bar{height:8px;background:#0b0d11;border:1px solid #1a1d23;border-radius:4px;overflow:hidden;margin-top:6px}
  .up-bar > div{height:100%;width:0;background:var(--acc);transition:width .15s ease}
  .up-bar.done > div{background:var(--ok)}
  .up-bar.error > div{background:var(--warn)}
  .up-total{margin-top:10px}
  .up-total .up-bar{height:10px}
  .modal-back{position:fixed;inset:0;background:rgba(0,0,0,.6);display:flex;align-items:center;justify-content:center;z-index:200}
  .modal{background:var(--panel);border:1px solid #222;border-radius:10px;max-width:min(92vw,700px);max-height:85vh;overflow:auto;padding:18px;min-width:320px}
  .modal h3{margin-top:0}
  .modal .actions{display:flex;gap:8px;justify-content:flex-end;margin-top:14px}
  .editor{width:100%;min-height:55vh;font-family:ui-monospace,monospace;font-size:13px;line-height:1.45;white-space:pre;overflow:auto;tab-size:4}
</style>
</head>
<body>
<header>
  <h1>🖥️ PC Overlay</h1>
  <div class="overlay-switch">
    <label data-i18n="ui.overlay">Overlay</label>
    <select id="overlaySelect" onchange="switchOverlay(this.value)"></select>
    <span id="overlayInfo" class="muted"></span>
  </div>
  <nav id="nav">
    <button data-tab="dash" data-i18n="ui.tab_dash" class="active">Dashboard</button>
    <button data-tab="proc" data-i18n="ui.tab_proc">Prozesse</button>
    <button data-tab="svc" data-i18n="ui.tab_svc">Dienste</button>
    <button data-tab="files" data-i18n="ui.tab_files">Dateien</button>
    <button data-tab="shell" data-i18n="ui.tab_shell">Shell</button>
    <button data-tab="ssh" data-i18n="ui.tab_ssh">SSH-Ziele</button>
    <button data-tab="settings" data-i18n="ui.tab_settings">Einstellungen</button>
  </nav>
</header>
<main>

<section id="tab-dash">
  <div class="card"><h3 data-i18n="ui.system">System</h3><div class="grid" id="dashGrid"></div></div>
  <div class="card">
    <div class="row" style="margin-bottom:12px">
      <h3 style="margin:0;flex:1" data-i18n="ui.storage">Speichermedien</h3>
      <span class="muted" id="diskCount"></span>
      <button class="act" onclick="toggleDiskEdit()" id="diskEditBtn" data-i18n="ui.edit_selection">Auswahl bearbeiten</button>
    </div>
    <div id="diskListGrid" class="grid"></div>
    <div id="diskEditPanel" class="hidden" style="margin-top:14px;padding-top:14px;border-top:1px solid #222">
      <div class="muted" style="margin-bottom:10px" data-i18n="ui.disk_hint"></div>
      <div id="diskEditList"></div>
      <div class="row" style="margin-top:12px">
        <button class="act primary" onclick="saveDiskVisibility()" data-i18n="ui.save_selection">Auswahl speichern</button>
        <button class="act" onclick="selectAllDisks()" data-i18n="ui.select_all">Alle auswählen</button>
        <button class="act" onclick="selectNoneDisks()" data-i18n="ui.select_none">Keine auswählen</button>
        <button class="act" onclick="resetDiskVisibility()" data-i18n="ui.default_only_big">Standard (nur ab 5 GiB)</button>
        <button class="act" onclick="toggleDiskEdit()" data-i18n="ui.cancel">Abbrechen</button>
      </div>
    </div>
  </div>
  <div class="card"><h3 data-i18n="ui.load_avg">Load Average (1 / 5 / 15 min)</h3><pre id="loadavg">–</pre></div>
</section>

<section id="tab-proc" class="hidden">
  <div class="card">
    <div class="row">
      <input id="procFilter" placeholder="Filter…">
      <button class="act primary" onclick="loadProcs()" data-i18n="ui.refresh">Aktualisieren</button>
    </div>
    <div style="overflow:auto;margin-top:12px">
      <table><thead><tr>
        <th data-i18n="ui.pid">PID</th><th data-i18n="ui.name">Name</th>
        <th data-i18n="ui.user">User</th><th data-i18n="ui.cpu_pct">CPU %</th>
        <th data-i18n="ui.ram_pct">RAM %</th><th data-i18n="ui.cmdline">Cmdline</th><th></th>
      </tr></thead>
      <tbody id="procBody"></tbody></table>
    </div>
  </div>
</section>

<section id="tab-svc" class="hidden">
  <div class="card">
    <div class="row">
      <input id="svcFilter" placeholder="Filter…">
      <button class="act primary" onclick="loadSvcs()" data-i18n="ui.refresh">Aktualisieren</button>
    </div>
    <div style="overflow:auto;margin-top:12px">
      <table><thead><tr>
        <th data-i18n="ui.name">Name</th><th data-i18n="ui.load">Load</th>
        <th data-i18n="ui.active">Active</th><th data-i18n="ui.sub">Sub</th><th></th>
      </tr></thead>
      <tbody id="svcBody"></tbody></table>
    </div>
  </div>
  <div class="card hidden" id="logCard">
    <div class="row">
      <h3 id="logTitle" style="margin:0;flex:1">Logs</h3>
      <button class="act" onclick="document.getElementById('logCard').classList.add('hidden')" data-i18n="ui.close">schließen</button>
    </div>
    <pre id="logOut" style="margin-top:12px"></pre>
  </div>
</section>

<section id="tab-files" class="hidden">
  <div class="card">
    <div class="row">
      <input id="filePath" value="/" placeholder="/path">
      <button class="act" onclick="navUp()" data-i18n="ui.parent">↑ Übergeordnet</button>
      <button class="act" onclick="refreshFiles()" data-i18n="ui.refresh">Aktualisieren</button>
      <button class="act primary" onclick="loadFiles()" data-i18n="ui.open">Öffnen</button>
    </div>

    <div id="sudoNotice" class="notice hidden" style="margin-top:12px" data-i18n="ui.sudo_missing"></div>
    <div id="sudoOkNotice" class="notice ok hidden" style="margin-top:12px" data-i18n="ui.sudo_ok"></div>

    <div class="split" style="margin-top:14px">
      <div id="uploadZone" class="drop">
        <input id="uploadInput" type="file" multiple style="display:none" onchange="uploadFiles(this.files)">
        <div><b data-i18n="ui.upload_title">Datei(en) hochladen</b></div>
        <div class="muted" style="margin-top:4px" data-i18n="ui.upload_hint"></div>
      </div>
      <div>
        <div class="muted" data-i18n="ui.target_dir">Aktueller Zielordner:</div>
        <pre id="uploadTarget" style="margin-top:6px;max-height:80px">/</pre>
      </div>
    </div>

    <div id="uploadProgressWrap" class="hidden">
      <div class="up-list" id="uploadList"></div>
      <div class="up-total" id="uploadTotalWrap">
        <div class="row" style="justify-content:space-between">
          <span class="muted" id="uploadTotalLabel" data-i18n="ui.upload_total">Gesamt</span>
          <span class="muted" id="uploadTotalPercent">0 %</span>
        </div>
        <div class="up-bar" id="uploadTotalBar"><div style="width:0%"></div></div>
      </div>
    </div>

    <div class="toolbar">
      <span class="sel" id="selInfo">0</span>
      <button class="act" onclick="selectAllFiles()" data-i18n="ui.select_all_btn">Alle auswählen</button>
      <button class="act" onclick="clearSelection()" data-i18n="ui.clear_selection">Auswahl leeren</button>
      <button class="act primary" onclick="downloadSelectionZip()" data-i18n="ui.zip_selection">Auswahl als ZIP</button>
      <button class="act" onclick="promptMkdir()" data-i18n="ui.new_folder">Neuer Ordner</button>
      <button class="act danger" onclick="deleteSelection()" data-i18n="ui.delete_selection">Auswahl löschen</button>
    </div>

    <div style="overflow:auto;margin-top:14px">
      <table>
        <thead><tr>
          <th style="width:28px"><input type="checkbox" id="selectAllBox" onchange="toggleAll(this.checked)"></th>
          <th data-i18n="ui.name">Name</th><th data-i18n="ui.perm">Rechte</th>
          <th data-i18n="ui.owner">Besitzer</th><th data-i18n="ui.size">Größe</th>
          <th data-i18n="ui.modified">Geändert</th><th></th>
        </tr></thead>
        <tbody id="fileBody"></tbody>
      </table>
    </div>
  </div>
  <div class="card hidden" id="fileView">
    <div class="row">
      <h3 id="fileViewName" style="margin:0;flex:1"></h3>
      <button class="act primary" onclick="saveEditor()" data-i18n="ui.save">Speichern</button>
      <button class="act" onclick="closeEditor()" data-i18n="ui.close">schließen</button>
    </div>
    <div class="muted" id="editorHint" style="margin-top:6px"></div>
    <textarea id="editorArea" class="editor" spellcheck="false" style="margin-top:12px"></textarea>
  </div>
</section>

<section id="tab-shell" class="hidden">
  <div class="card">
    <div class="row">
      <input id="shellCmd" placeholder="command…" autocomplete="off">
      <label class="muted" style="display:flex;gap:6px;align-items:center">
        <input type="checkbox" id="shellSudo"> <span data-i18n="ui.with_sudo">mit sudo</span>
      </label>
      <button class="act primary" onclick="runShell()" data-i18n="ui.run">Ausführen</button>
    </div>
    <pre id="shellOut" style="margin-top:12px" data-i18n="ui.ready">Bereit.</pre>
  </div>
</section>

<section id="tab-ssh" class="hidden">
  <div class="notice" data-i18n="ui.ssh_warning"></div>

  <div class="card">
    <h3 data-i18n="ui.add_edit_ssh">SSH-Ziel hinzufügen / bearbeiten</h3>
    <div class="split">
      <div><label class="field" data-i18n="ui.name">Name</label><input id="sshName" placeholder="my-server"></div>
      <div><label class="field" data-i18n="ui.host_ip">Host / IP</label><input id="sshHost" placeholder="192.168.1.100"></div>
      <div><label class="field" data-i18n="ui.port">Port</label><input id="sshPort" type="number" value="22" min="1" max="65535"></div>
      <div><label class="field" data-i18n="ui.user">User</label><input id="sshUser" placeholder="user"></div>
      <div>
        <label class="field" data-i18n="ui.auth_method">Auth-Methode</label>
        <select id="sshAuth" onchange="toggleAuthFields()">
          <option value="password" data-i18n="ui.auth_password">Passwort</option>
          <option value="key" data-i18n="ui.auth_key">SSH-Key</option>
        </select>
      </div>
    </div>
    <div id="authPasswordBlock" style="margin-top:12px">
      <label class="field" data-i18n="ui.ssh_password">SSH-Passwort (wird gespeichert)</label>
      <input id="sshPassword" type="password">
    </div>
    <div id="authKeyBlock" class="hidden" style="margin-top:12px">
      <label class="field" data-i18n="ui.ssh_key_path">Pfad zum privaten Key</label>
      <input id="sshKeyPath" placeholder="~/.ssh/id_rsa">
      <label class="field" style="margin-top:10px" data-i18n="ui.ssh_key_pass">Key-Passphrase (optional)</label>
      <input id="sshKeyPass" type="password" data-i18n-ph="ui.ssh_key_pass_ph">
    </div>
    <div style="margin-top:12px">
      <label class="field" data-i18n="ui.sudo_password">Sudo-Passwort (optional)</label>
      <input id="sshSudoPassword" type="password" data-i18n-ph="ui.sudo_password_ph">
      <label class="muted" style="display:flex;gap:6px;align-items:center;margin-top:6px">
        <input type="checkbox" id="sshClearSudo"> <span data-i18n="ui.clear_sudo">Sudo-Passwort löschen</span>
      </label>
    </div>
    <div class="row" style="margin-top:14px">
      <button class="act primary" onclick="saveSshTarget()" data-i18n="ui.save_test">Speichern & testen</button>
      <button class="act" onclick="clearSshForm()" data-i18n="ui.clear_form">Formular leeren</button>
    </div>
  </div>

  <div class="card">
    <div class="row">
      <h3 style="margin:0;flex:1" data-i18n="ui.stored_targets">Gespeicherte SSH-Ziele</h3>
      <button class="act" onclick="loadSshTargets()" data-i18n="ui.refresh">Aktualisieren</button>
    </div>
    <div style="overflow:auto;margin-top:12px">
      <table><thead><tr>
        <th data-i18n="ui.name">Name</th><th data-i18n="ui.host_ip">Host</th>
        <th data-i18n="ui.port">Port</th><th data-i18n="ui.user">User</th>
        <th data-i18n="ui.auth">Auth</th><th data-i18n="ui.sudo">Sudo</th><th></th>
      </tr></thead>
      <tbody id="sshBody"></tbody></table>
    </div>
  </div>
</section>

<section id="tab-settings" class="hidden">
  <div class="card">
    <h3 data-i18n="ui.current_config">Aktuelle Konfiguration</h3>
    <pre id="currentBind">–</pre>
  </div>
  <div class="card">
    <h3 data-i18n="ui.language">Sprache</h3>
    <select id="langSelect" style="width:100%">
      <option value="de">Deutsch</option>
      <option value="en">English</option>
      <option value="ru">Русский</option>
      <option value="ja">日本語</option>
      <option value="ko">한국어</option>
      <option value="zh">中文</option>
    </select>
  </div>
  <div class="card">
    <h3 data-i18n="ui.bind_addr_port">Bind-Adresse & Port</h3>
    <label class="field" data-i18n="ui.bind_ip">Bind-IP</label>
    <select id="bindSelect" style="width:100%"></select>
    <label class="field" style="margin-top:10px" data-i18n="ui.custom_ip">Oder eigene IP eingeben</label>
    <input id="bindCustom" placeholder="192.168.1.100" style="width:100%">
    <label class="field" style="margin-top:14px" data-i18n="ui.port">Port</label>
    <div class="row">
      <input id="portInput" type="number" min="1024" max="65535" style="flex:1">
      <button class="act" onclick="randomPort()" data-i18n="ui.random_port">Zufälliger Port</button>
      <button class="act" onclick="checkPort()" data-i18n="ui.check_port">Port prüfen</button>
    </div>
    <div id="portStatus" class="muted" style="margin-top:8px"></div>
  </div>
  <div class="card">
    <h3 data-i18n="ui.change_login">Login ändern (optional)</h3>
    <input id="newUser" data-i18n-ph="ui.new_user_ph" style="width:100%;margin-bottom:8px">
    <input id="newPass" type="password" data-i18n-ph="ui.new_pass_ph" style="width:100%;margin-bottom:8px">
    <input id="newPass2" type="password" data-i18n-ph="ui.new_pass2_ph" style="width:100%">
  </div>
  <div class="card">
    <div class="row">
      <label style="display:flex;gap:8px;align-items:center;cursor:pointer">
        <input type="checkbox" id="allowDelete">
        <span data-i18n="ui.allow_delete">Löschen im Datei-Browser erlauben</span>
      </label>
    </div>
  </div>
  <div class="card">
    <div class="row">
      <button class="act primary" onclick="saveSettings()" data-i18n="ui.save">Speichern</button>
      <button class="act" onclick="loadSettings()" data-i18n="ui.reset">Zurücksetzen</button>
    </div>
    <div class="muted" style="margin-top:10px" data-i18n="ui.restart_note"></div>
  </div>
</section>

</main>
<div class="toast" id="toast"></div>
<div id="modalHost"></div>

<script>
"use strict";
const I18N = __I18N_JSON__;
function t(key, vars){
  let s = (I18N.strings && I18N.strings[key]) || key;
  if (vars) for (const k of Object.keys(vars)) s = s.split("{"+k+"}").join(vars[k]);
  return s;
}
function applyI18n(){
  document.querySelectorAll("[data-i18n]").forEach(el => {
    const k = el.getAttribute("data-i18n");
    if (k) el.textContent = t(k);
  });
  document.querySelectorAll("[data-i18n-ph]").forEach(el => {
    const k = el.getAttribute("data-i18n-ph");
    if (k) el.placeholder = t(k);
  });
  // Placeholder für Filter-Felder dynamisch setzen
  const pf = document.getElementById("procFilter"); if (pf) pf.placeholder = t("ui.filter_cmd");
  const sf = document.getElementById("svcFilter"); if (sf) sf.placeholder = t("ui.filter");
  const fp = document.getElementById("filePath"); if (fp) fp.placeholder = t("ui.path");
  const sh = document.getElementById("shellCmd"); if (sh) sh.placeholder = t("ui.shell_placeholder");
}

const $ = id => document.getElementById(id);
let currentOverlay = 'local';
let overlayList = [];
let lastSystem = null;
let currentFileList = [];
let currentDir = '/';
let editorPath = null;
const selected = new Set();
const DEFAULT_MIN_AUTO = 5 * 1024 * 1024 * 1024;
let diskEditDirty = {};

const qs = (url) => {
  const sep = url.includes('?') ? '&' : '?';
  return url + sep + 'overlay=' + encodeURIComponent(currentOverlay);
};
const api = async (url, opts={}) => {
  const r = await fetch(qs(url), {credentials:'include', ...opts});
  const txt = await r.text();
  let data; try { data = JSON.parse(txt); } catch { data = {raw: txt}; }
  if (!r.ok) throw new Error(data.detail || data.raw || r.statusText);
  return data;
};
const apiRaw = async (url, opts={}) => {
  const r = await fetch(url, {credentials:'include', ...opts});
  const txt = await r.text();
  let data; try { data = JSON.parse(txt); } catch { data = {raw: txt}; }
  if (!r.ok) throw new Error(data.detail || data.raw || r.statusText);
  return data;
};
const fmtBytes = b => {
  if (b == null) return '–';
  const u=['B','KB','MB','GB','TB','PB']; let i=0;
  while (b>=1024 && i<u.length-1){b/=1024;i++;}
  return b.toFixed(1)+' '+u[i];
};
const fmtUptime = s => {
  const d=Math.floor(s/86400),h=Math.floor(s%86400/3600),m=Math.floor(s%3600/60);
  return d+'d '+h+'h '+m+'m';
};
const esc = s => String(s == null ? '' : s).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
let toastTimer;
function toast(msg){
  const el = $('toast'); el.textContent = msg; el.classList.add('show');
  clearTimeout(toastTimer); toastTimer = setTimeout(()=>el.classList.remove('show'), 4000);
}

function showModal({title, bodyHtml, okLabel, cancelLabel, danger=false,
                    onOk=null, showCancel=true}) {
  return new Promise(resolve => {
    const host = $('modalHost');
    const back = document.createElement('div');
    back.className = 'modal-back';
    back.innerHTML =
      '<div class="modal">' +
        '<h3>'+esc(title)+'</h3>' +
        '<div>'+bodyHtml+'</div>' +
        '<div class="actions">' +
          (showCancel ? '<button class="act" id="modalCancel">'+esc(cancelLabel||t("ui.cancel"))+'</button>' : '') +
          '<button class="act '+(danger?'danger':'primary')+'" id="modalOk">'+esc(okLabel||'OK')+'</button>' +
        '</div>' +
      '</div>';
    host.appendChild(back);
    const close = (result) => { back.remove(); resolve(result); };
    const cancelBtn = back.querySelector('#modalCancel');
    if (cancelBtn) cancelBtn.addEventListener('click', () => close(null));
    back.querySelector('#modalOk').addEventListener('click', async () => {
      if (onOk) {
        try {
          const r = await onOk(back);
          if (r !== undefined) close(r);
        } catch(e){ close(null); }
      } else close(true);
    });
    back.addEventListener('click', e => { if (e.target === back) close(null); });
  });
}
async function confirmModal(title, message, opts={}) {
  return showModal({title, bodyHtml: '<p>'+esc(message)+'</p>',
                    danger: !!opts.danger, okLabel: opts.okLabel});
}
async function promptModal(title, message, defaultValue='', opts={}) {
  return showModal({
    title, okLabel: opts.okLabel, danger: !!opts.danger,
    bodyHtml: '<p>'+esc(message)+'</p><input id="modalInput" style="width:100%;margin-top:8px" value="'+esc(defaultValue)+'">',
    onOk: (back) => back.querySelector('#modalInput').value
  });
}

async function loadOverlays(){
  try {
    const r = await apiRaw('/api/overlays');
    overlayList = r.overlays;
    const sel = $('overlaySelect');
    sel.innerHTML = overlayList.map(o =>
      '<option value="'+o.name+'">'+esc(o.label)+'</option>'
    ).join('');
    if (!overlayList.find(o => o.name === currentOverlay)) currentOverlay = 'local';
    sel.value = currentOverlay;
    updateOverlayInfo();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
function updateOverlayInfo(){
  const o = overlayList.find(x => x.name === currentOverlay);
  if (!o){ $('overlayInfo').innerHTML = ''; return; }
  if (o.type === 'local'){
    $('overlayInfo').innerHTML = esc(t("ui.overlay_local"));
  } else {
    const sudoTag = o.has_sudo ? ' <span class="tag sudo">sudo</span>' : '';
    $('overlayInfo').innerHTML =
      '<span class="tag ssh">SSH</span> ' + esc(o.user) + '@' + esc(o.host) + ':' + o.port + sudoTag;
  }
}
function switchOverlay(name){
  currentOverlay = name;
  updateOverlayInfo();
  $('diskEditPanel').classList.add('hidden');
  $('diskEditBtn').textContent = t("ui.edit_selection");
  document.body.classList.remove('disk-edit-active');
  diskEditDirty = {};
  selected.clear();
  updateSelInfo();
  const active = document.querySelector('#nav button.active');
  const tab = active ? active.dataset.tab : 'dash';
  reloadTab(tab);
}
function reloadTab(tab){
  if (tab==='dash') loadDash();
  else if (tab==='proc') loadProcs();
  else if (tab==='svc')  loadSvcs();
  else if (tab==='files') loadFiles();
  else if (tab==='ssh')    loadSshTargets();
  else if (tab==='settings') loadSettings();
}

document.querySelectorAll('#nav button').forEach(b => {
  b.onclick = () => {
    document.querySelectorAll('#nav button').forEach(x=>x.classList.remove('active'));
    b.classList.add('active');
    ['dash','proc','svc','files','shell','ssh','settings'].forEach(tab =>
      $('tab-'+tab).classList.toggle('hidden', tab !== b.dataset.tab));
    reloadTab(b.dataset.tab);
  };
});

async function loadDash(){
  try {
    const s = await api('/api/system');
    lastSystem = s;
    $('dashGrid').innerHTML =
      '<div class="stat"><span>'+esc(t("ui.overlay"))+'</span><b style="font-size:14px">'+esc(s.overlay)+'</b></div>' +
      '<div class="stat"><span>'+esc(t("ui.hostname"))+'</span><b>'+esc(s.hostname)+'</b></div>' +
      '<div class="stat"><span>'+esc(t("ui.os"))+'</span><b style="font-size:14px">'+esc(s.platform)+'</b></div>' +
      '<div class="stat"><span>'+esc(t("ui.python"))+'</span><b style="font-size:14px">'+esc(s.python)+'</b></div>' +
      '<div class="stat"><span>'+esc(t("ui.uptime"))+'</span><b>'+fmtUptime(s.uptime_sec)+'</b></div>' +
      '<div class="stat"><span>'+esc(t("ui.cpu_usage"))+'</span><b>'+s.cpu_percent.toFixed(1)+' %</b><span style="color:#888">'+s.cpu_count+' '+esc(t("ui.cores"))+'</span></div>' +
      '<div class="stat"><span>'+esc(t("ui.ram"))+'</span><b>'+s.mem_percent.toFixed(1)+' %</b><span style="color:#888">'+fmtBytes(s.mem_used)+' / '+fmtBytes(s.mem_total)+'</span></div>' +
      '<div class="stat"><span>'+esc(t("ui.bind_port"))+'</span><b style="font-size:14px">'+esc(s.bind_ip)+':'+s.port+'</b></div>';
    renderDiskList();
    if (!$('diskEditPanel').classList.contains('hidden')) renderDiskEditList();
    $('loadavg').textContent = s.load_avg.map(x=>Number(x).toFixed(2)).join('   ');
  } catch(e) { toast(t("ui.error", {err: e.message})); }
}
setInterval(()=>{
  if (document.body.classList.contains('disk-edit-active')) return;
  if ($('tab-dash').classList.contains('hidden')) return;
  loadDash();
}, 3000);

function renderDiskList(){
  if (!lastSystem) return;
  const disks = Array.isArray(lastSystem.disks) ? lastSystem.disks : [];
  const autoBytes = lastSystem.auto_filter_bytes || DEFAULT_MIN_AUTO;
  const visible = lastSystem.visible_disks;
  const modeTag = (visible === null)
    ? '<span class="tag auto">'+esc(t("ui.auto_ge", {size: fmtBytes(autoBytes)}))+'</span>'
    : '<span class="tag">'+esc(t("ui.manual"))+'</span>';
  $('diskCount').innerHTML = disks.length + ' ' + esc(t("ui.shown")) + ' ' + modeTag;
  if (disks.length === 0){
    $('diskListGrid').innerHTML =
      '<div class="muted">'+esc(t("ui.no_disks"))+'</div>';
    return;
  }
  $('diskListGrid').innerHTML = disks.map(d => {
    const label = d.mountpoint === '/' ? (t("ui.disk")+' /') : (t("ui.disk")+' ' + d.mountpoint);
    const sub = (d.device || '') + (d.fstype ? ' · ' + d.fstype : '');
    const smallTag = d.small ? ' <span class="tag small">'+esc(t("ui.small"))+'</span>' : '';
    return '<div class="stat">' +
      '<span>'+esc(label)+smallTag+'</span>' +
      '<b>'+d.percent.toFixed(1)+' %</b>' +
      '<span style="color:#888">'+fmtBytes(d.used)+' / '+fmtBytes(d.total)+'</span>' +
      '<span style="color:#666;font-size:11px;display:block;margin-top:4px;word-break:break-all">'+esc(sub)+'</span>' +
      '<button class="act" style="margin-top:6px" onclick="openDiskInFiles(\''+esc(d.mountpoint).replace(/'/g,"\\'")+'\')">'+esc(t("ui.open_in_files"))+'</button>' +
    '</div>';
  }).join('');
}
function openDiskInFiles(mount){
  document.querySelectorAll('#nav button').forEach(x => x.classList.toggle('active', x.dataset.tab === 'files'));
  ['dash','proc','svc','files','shell','ssh','settings'].forEach(tab =>
    $('tab-'+tab).classList.toggle('hidden', tab !== 'files'));
  $('filePath').value = mount;
  loadFiles();
}
function toggleDiskEdit(){
  const panel = $('diskEditPanel');
  const btn = $('diskEditBtn');
  const showing = !panel.classList.contains('hidden');
  if (showing){
    panel.classList.add('hidden');
    btn.textContent = t("ui.edit_selection");
    document.body.classList.remove('disk-edit-active');
  } else {
    panel.classList.remove('hidden');
    btn.textContent = t("ui.close_selection");
    document.body.classList.add('disk-edit-active');
    renderDiskEditList();
  }
}
function readDiskEditDOM(){
  document.querySelectorAll('#diskEditList input[type=checkbox]').forEach(cb => {
    diskEditDirty[cb.dataset.mount] = cb.checked;
  });
}
function renderDiskEditList(){
  if (!lastSystem) return;
  if (Object.keys(diskEditDirty).length) readDiskEditDOM();

  const all = Array.isArray(lastSystem.all_disks) ? lastSystem.all_disks : [];
  const autoBytes = lastSystem.auto_filter_bytes || DEFAULT_MIN_AUTO;
  const visible = lastSystem.visible_disks;
  if (all.length === 0){
    $('diskEditList').innerHTML = '<div class="muted">'+esc(t("ui.no_disks"))+'</div>';
    return;
  }
  $('diskEditList').innerHTML = all.map(d => {
    let checked;
    if (d.mountpoint in diskEditDirty)     checked = diskEditDirty[d.mountpoint];
    else if (visible === null)             checked = !d.small;
    else                                   checked = visible.includes(d.mountpoint);
    const sub = (d.device || '') + (d.fstype ? ' · ' + d.fstype : '');
    const sizeTag = d.small
      ? '<span class="tag small">'+esc(t("ui.small"))+' (< '+fmtBytes(autoBytes)+')</span>' : '';
    return '<div class="disk-row">' +
      '<input type="checkbox" data-mount="'+esc(d.mountpoint)+'" '+(checked?'checked':'')+' ' +
        'onchange="diskEditDirty[this.dataset.mount]=this.checked">' +
      '<div class="disk-label">' +
        '<div><b>'+esc(d.mountpoint)+'</b> '+sizeTag+' <span class="muted">'+esc(sub)+'</span></div>' +
        '<div class="muted">'+fmtBytes(d.used)+' / '+fmtBytes(d.total)+' – '+d.percent.toFixed(1)+' %</div>' +
      '</div>' +
      '<div class="disk-bar"><div style="width:'+Math.min(100,d.percent).toFixed(1)+'%"></div></div>' +
    '</div>';
  }).join('');
}
function selectAllDisks(){
  document.querySelectorAll('#diskEditList input[type=checkbox]').forEach(c => {
    c.checked = true; diskEditDirty[c.dataset.mount] = true;
  });
}
function selectNoneDisks(){
  document.querySelectorAll('#diskEditList input[type=checkbox]').forEach(c => {
    c.checked = false; diskEditDirty[c.dataset.mount] = false;
  });
}
async function saveDiskVisibility(){
  const boxes = [...document.querySelectorAll('#diskEditList input[type=checkbox]')];
  const sel = boxes.filter(b => b.checked).map(b => b.dataset.mount);
  const all = Array.isArray(lastSystem.all_disks) ? lastSystem.all_disks : [];
  const autoBytes = lastSystem.auto_filter_bytes || DEFAULT_MIN_AUTO;
  const autoMounts = all.filter(d => d.total >= autoBytes).map(d => d.mountpoint);
  const sameAsAuto = (sel.length === autoMounts.length)
                   && autoMounts.every(m => sel.includes(m));
  const mode = sameAsAuto ? 'auto' : 'custom';
  try {
    await api('/api/disks/visibility', {
      method: 'POST', headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({mode, mounts: sel})
    });
    diskEditDirty = {};
    toast(mode === 'auto' ? t("ui.default_saved") : t("ui.selection_saved"));
    await loadDash();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function resetDiskVisibility(){
  try {
    await api('/api/disks/visibility', {
      method: 'POST', headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({mode: 'auto', mounts: []})
    });
    diskEditDirty = {};
    toast(t("ui.default_restored"));
    await loadDash();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}

async function loadProcs(){
  try {
    const list = await api('/api/processes');
    const f = ($('procFilter').value||'').toLowerCase();
    $('procBody').innerHTML = list
      .filter(p => !f || (p.name||'').toLowerCase().includes(f) || (p.cmdline||'').toLowerCase().includes(f))
      .map(p => '<tr>' +
        '<td>'+p.pid+'</td>' +
        '<td><b>'+esc(p.name)+'</b></td>' +
        '<td>'+esc(p.username||'')+'</td>' +
        '<td>'+(p.cpu_percent||0).toFixed(1)+'</td>' +
        '<td>'+(p.memory_percent||0).toFixed(1)+'</td>' +
        '<td class="muted" style="max-width:320px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap">'+esc(p.cmdline||'')+'</td>' +
        '<td><button class="act danger" onclick="killProc('+p.pid+')">'+esc(t("ui.kill"))+'</button></td>' +
      '</tr>').join('') || '<tr><td colspan="7" class="muted">'+esc(t("ui.no_procs"))+'</td></tr>';
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function killProc(pid){
  const ok = await confirmModal(t("ui.kill_confirm_title", {pid}), t("ui.kill_confirm_msg"),
                                {danger:true, okLabel: t("ui.kill_ok_label")});
  if (!ok) return;
  try { await api('/api/processes/'+pid+'/kill',{method:'POST'}); toast(t("ui.killed", {pid})); loadProcs(); }
  catch(e){ toast(t("ui.error", {err: e.message})); }
}
$('procFilter').oninput = loadProcs;

async function loadSvcs(){
  try {
    const list = await api('/api/services');
    const f = ($('svcFilter').value||'').toLowerCase();
    $('svcBody').innerHTML = list
      .filter(s => !f || s.name.toLowerCase().includes(f))
      .map(s => {
        const cls = s.active==='active' ? 'active' : (s.active==='failed' ? 'failed' : 'inactive');
        return '<tr>' +
          '<td>'+esc(s.name)+'</td><td>'+esc(s.load)+'</td>' +
          '<td><span class="tag '+cls+'">'+esc(s.active)+'</span></td>' +
          '<td>'+esc(s.sub)+'</td>' +
          '<td>' +
            '<button class="act" onclick="svcAct(\''+esc(s.name)+'\',\'start\')">'+esc(t("ui.start"))+'</button>' +
            '<button class="act" onclick="svcAct(\''+esc(s.name)+'\',\'stop\')">'+esc(t("ui.stop"))+'</button>' +
            '<button class="act" onclick="svcAct(\''+esc(s.name)+'\',\'restart\')">'+esc(t("ui.restart"))+'</button>' +
            '<button class="act" onclick="svcLogs(\''+esc(s.name)+'\')">'+esc(t("ui.logs"))+'</button>' +
          '</td>' +
        '</tr>';
      }).join('') || '<tr><td colspan="5" class="muted">'+esc(t("ui.no_services"))+'</td></tr>';
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function svcAct(name, action){
  try {
    const r = await api('/api/services/'+encodeURIComponent(name)+'/'+action,{method:'POST'});
    if (r.returncode !== 0) toast((r.stderr||'').slice(0,200) || t("ui.error", {err:'?'}));
    else toast(name+': '+action+' ok');
    setTimeout(loadSvcs, 500);
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function svcLogs(name){
  try {
    const r = await api('/api/services/'+encodeURIComponent(name)+'/logs?lines=400');
    $('logCard').classList.remove('hidden');
    $('logTitle').textContent = t("ui.logs_title", {name});
    $('logOut').textContent = r.output || t("ui.empty");
    $('logCard').scrollIntoView({behavior:'smooth',block:'start'});
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
$('svcFilter').oninput = loadSvcs;

async function loadFiles(){
  try {
    const path = $('filePath').value || '/';
    const r = await api('/api/files?path='+encodeURIComponent(path));
    closeEditor();
    if (r.type === 'file'){
      await openEditorFor(r.path);
      return;
    }
    currentDir = r.path;
    $('filePath').value = r.path;
    $('uploadTarget').textContent = r.path;
    currentFileList = r.entries.map(e => {
      const full = (r.path.replace(/\/$/,'') + '/' + e.name);
      return {...e, path: full};
    });
    const existingPaths = new Set(currentFileList.map(e => e.path));
    for (const p of [...selected]) if (!existingPaths.has(p)) selected.delete(p);
    renderFileTable();
    updateSelInfo();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
function refreshFiles(){ loadFiles(); }

function renderFileTable(){
  const rows = currentFileList.map(e => {
    const safe = e.path.replace(/\\/g,'\\\\').replace(/'/g,"\\'");
    const checked = selected.has(e.path) ? 'checked' : '';
    const icon = e.is_dir ? '📁' : '📄';
    const mode = e.mode || '';
    const owner = e.owner ? (e.owner + (e.group ? ':' + e.group : '')) : (e.uid != null ? String(e.uid) : '');
    return '<tr>' +
      '<td><input type="checkbox" data-path="'+esc(e.path)+'" '+checked+' onchange="toggleSelect(this)"></td>' +
      '<td style="cursor:pointer" onclick="openEntry(\''+safe+'\')">'+icon+' '+esc(e.name)+'</td>' +
      '<td class="muted">'+esc(mode)+'</td>' +
      '<td class="muted">'+esc(owner)+'</td>' +
      '<td>'+(e.is_dir?'–':fmtBytes(e.size))+'</td>' +
      '<td class="muted">'+new Date(e.mtime*1000).toLocaleString()+'</td>' +
      '<td>' +
        (e.is_dir
          ? '<button class="act" onclick="event.stopPropagation();deleteEntry(\''+safe+'\')">'+esc(t("ui.delete"))+'</button>'
          : '<button class="act" onclick="event.stopPropagation();editFile(\''+safe+'\')">'+esc(t("ui.edit"))+'</button>' +
            '<button class="act" onclick="event.stopPropagation();downloadFile(\''+safe+'\')">'+esc(t("ui.download"))+'</button>' +
            '<button class="act" onclick="event.stopPropagation();deleteEntry(\''+safe+'\')">'+esc(t("ui.delete"))+'</button>') +
        '<button class="act" onclick="event.stopPropagation();renameEntry(\''+safe+'\')">'+esc(t("ui.rename"))+'</button>' +
        '<button class="act" onclick="event.stopPropagation();chmodQuick(\''+safe+'\')">'+esc(t("ui.perms"))+'</button>' +
        '<button class="act" onclick="event.stopPropagation();chownEntry(\''+safe+'\')">'+esc(t("ui.chown"))+'</button>' +
      '</td>' +
    '</tr>';
  }).join('');
  $('fileBody').innerHTML = rows || '<tr><td colspan="7" class="muted">'+esc(t("ui.empty"))+'</td></tr>';
  $('selectAllBox').checked = currentFileList.length > 0 && currentFileList.every(e => selected.has(e.path));
}
function toggleSelect(cb){
  const p = cb.dataset.path;
  if (cb.checked) selected.add(p); else selected.delete(p);
  updateSelInfo();
}
function toggleAll(checked){
  if (checked) currentFileList.forEach(e => selected.add(e.path));
  else selected.clear();
  renderFileTable();
  updateSelInfo();
}
function selectAllFiles(){ currentFileList.forEach(e => selected.add(e.path)); renderFileTable(); updateSelInfo(); }
function clearSelection(){ selected.clear(); renderFileTable(); updateSelInfo(); }
function updateSelInfo(){ $('selInfo').textContent = t("ui.selected_n", {n: selected.size}); }
function openEntry(p){ $('filePath').value = p; loadFiles(); }
function downloadFile(p){ window.location.href = qs('/api/files/download?path=' + encodeURIComponent(p)); }
function navUp(){
  let p = $('filePath').value.replace(/\/+$/,'');
  if (!p) p = '/';
  const idx = p.lastIndexOf('/');
  $('filePath').value = idx <= 0 ? '/' : p.slice(0, idx);
  loadFiles();
}
$('filePath').addEventListener('keydown', e=>{ if(e.key==='Enter') loadFiles(); });

async function editFile(p){ await openEditorFor(p); }
async function openEditorFor(p){
  try {
    const r = await api('/api/files/edit?path=' + encodeURIComponent(p));
    editorPath = r.path;
    $('fileView').classList.remove('hidden');
    $('fileViewName').textContent = r.path;
    $('editorHint').textContent = r.size + ' B';
    $('editorArea').value = r.content;
    $('fileView').scrollIntoView({behavior:'smooth',block:'start'});
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
function closeEditor(){
  editorPath = null;
  $('fileView').classList.add('hidden');
}
async function saveEditor(){
  if (!editorPath) return;
  try {
    await api('/api/files/save', {
      method: 'POST', headers: {'Content-Type':'application/json'},
      body: JSON.stringify({path: editorPath, content: $('editorArea').value})
    });
    toast(t("ui.saved"));
    $('editorHint').textContent = $('editorArea').value.length + ' B';
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}

async function downloadSelectionZip(){
  if (selected.size === 0){ toast(t("ui.no_selection")); return; }
  try {
    const r = await fetch(qs('/api/files/download_zip'), {
      method: 'POST', credentials: 'include',
      headers: {'Content-Type':'application/json'},
      body: JSON.stringify({paths: [...selected]})
    });
    if (!r.ok){ throw new Error(await r.text()); }
    const blob = await r.blob();
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = 'pcoverlay-' + Date.now() + '.zip';
    document.body.appendChild(a); a.click(); a.remove();
    setTimeout(() => URL.revokeObjectURL(url), 10000);
    toast(t("ui.zip_created"));
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function deleteSelection(){
  if (selected.size === 0){ toast(t("ui.no_selection")); return; }
  const n = selected.size;
  const ok = await confirmModal(t("ui.delete_confirm", {n}), t("ui.delete_recursive"),
                                {danger:true, okLabel: t("ui.delete_ok_label")});
  if (!ok) return;
  try {
    const r = await api('/api/files/delete', {
      method: 'POST', headers: {'Content-Type':'application/json'},
      body: JSON.stringify({paths: [...selected]})
    });
    const failed = (r.results || []).filter(x => !x.ok);
    if (failed.length === 0) toast(t("ui.deleted_n", {n}));
    else toast(t("ui.deleted_failed", {failed: failed.length, total: n}));
    selected.clear();
    loadFiles();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function deleteEntry(p){
  const ok = await confirmModal(t("ui.delete"), p, {danger:true, okLabel: t("ui.delete_ok_label")});
  if (!ok) return;
  try {
    const r = await api('/api/files/delete', {
      method: 'POST', headers: {'Content-Type':'application/json'},
      body: JSON.stringify({paths: [p]})
    });
    if (r.ok) toast(t("ui.deleted_one")); else toast(t("ui.delete_error"));
    selected.delete(p);
    loadFiles();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function renameEntry(p){
  const oldName = p.split('/').pop();
  const newName = await promptModal(t("ui.rename_title"), t("ui.rename_prompt", {name: oldName}), oldName,
                                    {okLabel: t("ui.rename_ok_label")});
  if (!newName || newName === oldName) return;
  try {
    await api('/api/files/rename', {
      method: 'POST', headers: {'Content-Type':'application/json'},
      body: JSON.stringify({path: p, new_name: newName})
    });
    toast(t("ui.renamed"));
    loadFiles();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function chmodQuick(p){
  const mode = await showModal({
    title: t("ui.perms_title"),
    bodyHtml: '<p class="muted">'+esc(p)+'</p>' +
      '<input id="modalInput" style="width:100%;margin-top:8px" value="644">' +
      '<div class="row" style="margin-top:10px">' +
      ['644','600','640','755','700','777'].map(m =>
        '<button class="act" type="button" onclick="document.getElementById(\'modalInput\').value=\''+m+'\'">'+m+'</button>'
      ).join('') +
      '</div>',
    okLabel: t("ui.perms_ok_label"),
    onOk: (back) => back.querySelector('#modalInput').value
  });
  if (!mode) return;
  const modeStr = String(mode).trim();
  if (!/^[0-7]{3,4}$/.test(modeStr)) { toast(t("ui.invalid_mode")); return; }
  try {
    await api('/api/files/chmod', {
      method: 'POST', headers: {'Content-Type':'application/json'},
      body: JSON.stringify({path: p, mode: modeStr})
    });
    toast(t("ui.perms_set"));
    loadFiles();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function chownEntry(p){
  const owner = await promptModal(t("ui.chown_title"), t("ui.chown_prompt", {path: p}), 'root:root',
                                  {okLabel: t("ui.set")});
  if (!owner) return;
  try {
    await api('/api/files/chown', {
      method: 'POST', headers: {'Content-Type':'application/json'},
      body: JSON.stringify({path: p, owner})
    });
    toast(t("ui.owner_set"));
    loadFiles();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function promptMkdir(){
  const name = await promptModal(t("ui.new_folder_title", {dir: currentDir}), t("ui.new_folder_prompt"), '',
                                 {okLabel: t("ui.create_label")});
  if (!name) return;
  try {
    await api('/api/files/mkdir', {
      method: 'POST', headers: {'Content-Type':'application/json'},
      body: JSON.stringify({path: currentDir, name})
    });
    toast(t("ui.folder_created"));
    loadFiles();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}

const zone = $('uploadZone');
zone.addEventListener('click', () => $('uploadInput').click());
zone.addEventListener('dragover', e => { e.preventDefault(); zone.classList.add('over'); });
zone.addEventListener('dragleave', () => zone.classList.remove('over'));
zone.addEventListener('drop', e => {
  e.preventDefault(); zone.classList.remove('over');
  if (e.dataTransfer.files && e.dataTransfer.files.length) uploadFiles(e.dataTransfer.files);
});
function upInit(totalFiles){
  $('uploadProgressWrap').classList.remove('hidden');
  $('uploadList').innerHTML = '';
  $('uploadTotalLabel').textContent = t("ui.upload_total") + ' (0 / ' + totalFiles + ')';
  $('uploadTotalPercent').textContent = '0 %';
  $('uploadTotalBar').className = 'up-bar';
  $('uploadTotalBar').firstElementChild.style.width = '0%';
}
function upAddItem(idx, name, size){
  const el = document.createElement('div');
  el.className = 'up-item';
  el.id = 'up-item-' + idx;
  el.innerHTML =
    '<div class="up-head">' +
      '<span class="up-name" title="'+esc(name)+'">'+esc(name)+'</span>' +
      '<span class="up-percent" id="up-pct-'+idx+'">0 %</span>' +
    '</div>' +
    '<div class="up-bar" id="up-bar-'+idx+'"><div style="width:0%"></div></div>' +
    '<div class="up-detail" id="up-detail-'+idx+'">0 B / '+fmtBytes(size)+'</div>';
  $('uploadList').appendChild(el);
}
function upSetProgress(idx, sent, total, phase){
  const pct = total > 0 ? Math.min(100, (sent / total) * 100) : 0;
  const pctEl = $('up-pct-'+idx); if (pctEl) pctEl.textContent = pct.toFixed(1) + ' %';
  const barEl = $('up-bar-'+idx); if (barEl) barEl.firstElementChild.style.width = pct.toFixed(2) + '%';
  const detailEl = $('up-detail-'+idx);
  if (detailEl){
    let detail = fmtBytes(sent) + ' / ' + fmtBytes(total);
    if (phase) detail += ' · ' + phase;
    detailEl.textContent = detail;
  }
}
function upSetState(idx, state){
  const barEl = $('up-bar-'+idx); if (!barEl) return;
  barEl.classList.remove('done', 'error');
  if (state === 'done') barEl.classList.add('done');
  if (state === 'error') barEl.classList.add('error');
}
function upUpdateTotal(doneCount, totalCount, pct){
  $('uploadTotalLabel').textContent = t("ui.upload_total") + ' (' + doneCount + ' / ' + totalCount + ')';
  $('uploadTotalPercent').textContent = pct.toFixed(1) + ' %';
  $('uploadTotalBar').firstElementChild.style.width = pct.toFixed(2) + '%';
  if (pct >= 100) $('uploadTotalBar').classList.add('done');
}
function uploadSingle(file, targetDir, idx, onProgress){
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    const fd = new FormData();
    fd.append('file', file);
    fd.append('path', targetDir);
    fd.append('overlay', currentOverlay);
    xhr.upload.onprogress = (ev) => { if (ev.lengthComputable) onProgress(ev.loaded, ev.total, null); };
    xhr.upload.onload = () => { onProgress(file.size, file.size, t("ui.upload_processing")); };
    xhr.onload = () => {
      let data; try { data = JSON.parse(xhr.responseText); } catch { data = {raw: xhr.responseText}; }
      if (xhr.status >= 200 && xhr.status < 300) resolve(data);
      else reject(new Error(data.detail || data.raw || ('HTTP ' + xhr.status)));
    };
    xhr.onerror = () => reject(new Error('network error'));
    xhr.open('POST', '/api/files/upload', true);
    xhr.withCredentials = true;
    xhr.send(fd);
  });
}
async function uploadFiles(files){
  if (!files || files.length === 0) return;
  const list = Array.from(files);
  const targetDir = $('uploadTarget').textContent.trim() || '/';
  const totalBytes = list.reduce((s, f) => s + f.size, 0);
  const sizeByIdx = list.map(f => f.size);
  const loadedByIdx = new Array(list.length).fill(0);
  let doneCount = 0;
  upInit(list.length);
  list.forEach((f, i) => upAddItem(i, f.name, f.size));
  const recomputeTotal = () => {
    const loaded = loadedByIdx.reduce((a, b) => a + b, 0);
    const pct = totalBytes > 0 ? (loaded / totalBytes) * 100 : 0;
    upUpdateTotal(doneCount, list.length, pct);
  };
  recomputeTotal();
  for (let i = 0; i < list.length; i++){
    const f = list[i];
    try {
      const res = await uploadSingle(f, targetDir, i, (sent, total, phase) => {
        loadedByIdx[i] = sent;
        upSetProgress(i, sent, total, phase);
        recomputeTotal();
      });
      loadedByIdx[i] = sizeByIdx[i];
      upSetProgress(i, sizeByIdx[i], sizeByIdx[i], res.mode === 'sudo' ? t("ui.upload_via_sudo") : null);
      upSetState(i, 'done');
      doneCount++;
      recomputeTotal();
    } catch(e){
      upSetState(i, 'error');
      upSetProgress(i, loadedByIdx[i], sizeByIdx[i], t("ui.upload_error") + ' ' + e.message);
      toast(t("ui.upload_failed", {name: f.name, err: e.message}));
      break;
    }
  }
  $('uploadInput').value = '';
  loadFiles();
}

async function runShell(){
  const cmd = $('shellCmd').value.trim();
  if (!cmd) return;
  const out = $('shellOut');
  const useSudo = $('shellSudo').checked;
  const target = currentOverlay === 'local' ? 'local' : 'SSH:' + currentOverlay;
  out.innerHTML = '<span class="spin"></span> ' + esc(t("ui.shell_running", {target})) + ' ' + esc(cmd);
  try {
    const r = await api('/api/shell',{
      method:'POST', headers:{'Content-Type':'application/json'},
      body: JSON.stringify({cmd, sudo: useSudo})
    });
    out.textContent = '$ ' + (useSudo ? 'sudo ' : '') + cmd + '\n['+target+' | exit ' + r.returncode + ']\n' +
      (r.stdout || '') + (r.stderr ? '\n[stderr]\n'+r.stderr : '');
  } catch(e){ out.textContent = t("ui.error", {err: e.message}); }
}
$('shellCmd').addEventListener('keydown', e=>{ if(e.key==='Enter') runShell(); });

function toggleAuthFields(){
  const auth = $('sshAuth').value;
  $('authPasswordBlock').classList.toggle('hidden', auth !== 'password');
  $('authKeyBlock').classList.toggle('hidden', auth !== 'key');
}
function clearSshForm(){
  ['sshName','sshHost','sshUser','sshPassword','sshKeyPath','sshKeyPass','sshSudoPassword']
    .forEach(id => $(id).value = '');
  $('sshPort').value = 22;
  $('sshAuth').value = 'password';
  $('sshClearSudo').checked = false;
  toggleAuthFields();
}
async function loadSshTargets(){
  try {
    const list = await apiRaw('/api/ssh_targets');
    $('sshBody').innerHTML = list.map(t_ => {
      const sudoTag = t_.sudo_password ? '<span class="tag sudo">'+esc(t("ui.yes"))+'</span>'
                                       : '<span class="tag inactive">'+esc(t("ui.no"))+'</span>';
      return '<tr>' +
        '<td><b>'+esc(t_.name)+'</b></td>' +
        '<td>'+esc(t_.host)+'</td>' +
        '<td>'+t_.port+'</td>' +
        '<td>'+esc(t_.user)+'</td>' +
        '<td><span class="tag ssh">'+esc(t_.auth||'password')+'</span></td>' +
        '<td>'+sudoTag+'</td>' +
        '<td>' +
          '<button class="act" onclick="testSshTarget(\''+esc(t_.name)+'\')">'+esc(t("ui.test"))+'</button>' +
          '<button class="act" onclick="editSshTarget(\''+esc(t_.name)+'\')">'+esc(t("ui.edit_btn"))+'</button>' +
          '<button class="act" onclick="promptSudoPassword(\''+esc(t_.name)+'\')">'+esc(t("ui.sudo_password_btn"))+'</button>' +
          '<button class="act danger" onclick="delSshTarget(\''+esc(t_.name)+'\')">'+esc(t("ui.delete_btn"))+'</button>' +
        '</td>' +
      '</tr>';
    }).join('') || '<tr><td colspan="7" class="muted">'+esc(t("ui.no_targets"))+'</td></tr>';
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function editSshTarget(name){
  try {
    const list = await apiRaw('/api/ssh_targets');
    const t_ = list.find(x => x.name === name);
    if (!t_) return;
    $('sshName').value = t_.name; $('sshHost').value = t_.host;
    $('sshPort').value = t_.port; $('sshUser').value = t_.user;
    $('sshAuth').value = t_.auth || 'password';
    $('sshPassword').value = t_.password || '';
    $('sshKeyPath').value = t_.key_path || '';
    $('sshKeyPass').value = t_.key_pass || '';
    $('sshSudoPassword').value = t_.sudo_password || '';
    $('sshClearSudo').checked = false;
    toggleAuthFields();
    toast(t("ui.form_loaded"));
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function saveSshTarget(){
  const body = {
    name: $('sshName').value.trim(), host: $('sshHost').value.trim(),
    port: parseInt($('sshPort').value, 10) || 22,
    user: $('sshUser').value.trim(), auth: $('sshAuth').value,
    password: $('sshPassword').value,
    key_path: $('sshKeyPath').value.trim(), key_pass: $('sshKeyPass').value,
    sudo_password: $('sshSudoPassword').value,
    clear_sudo: $('sshClearSudo').checked,
  };
  if (!body.name || !body.host || !body.user){ toast(t("ui.fields_required")); return; }
  try {
    const r = await apiRaw('/api/ssh_targets', {
      method: 'POST', headers: {'Content-Type': 'application/json'},
      body: JSON.stringify(body)
    });
    toast(t("ui.saved_ssh", {msg: (r.test ? r.test.message : '?')}));
    await loadSshTargets();
    await loadOverlays();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function delSshTarget(name){
  const ok = await confirmModal(t("ui.target_delete_title"), name,
                                {danger:true, okLabel: t("ui.delete_ok_label")});
  if (!ok) return;
  try {
    await apiRaw('/api/ssh_targets/'+encodeURIComponent(name), {method:'DELETE'});
    toast(t("ui.target_deleted"));
    if (currentOverlay === name) currentOverlay = 'local';
    await loadSshTargets();
    await loadOverlays();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function testSshTarget(name){
  try {
    const r = await apiRaw('/api/ssh_targets/'+encodeURIComponent(name)+'/test', {method:'POST'});
    toast((r.ok ? '✓ ' : '✗ ') + r.message);
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
async function promptSudoPassword(name){
  const pw = await promptModal(t("ui.sudo_pw_title"), t("ui.sudo_pw_prompt", {name}), '',
                               {okLabel: t("ui.set")});
  if (!pw) return;
  try {
    const r = await apiRaw('/api/ssh_targets/'+encodeURIComponent(name)+'/sudo_password', {
      method: 'POST', headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({password: pw})
    });
    toast((r.ok ? '✓ ' : '✗ ') + r.message);
    await loadSshTargets();
    await loadOverlays();
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}

async function loadSettings(){
  try {
    const s = await apiRaw('/api/settings');
    $('currentBind').textContent =
      t("ui.bind_ip") + ': ' + s.bind_ip + '\n' + t("ui.port") + ': ' + s.port + '\n' + t("ui.user") + ': ' + s.username;
    const sel = $('bindSelect');
    sel.innerHTML = s.available_ips.map(i =>
      '<option value="' + i.ip + '">' + i.ip + '  (' + i.iface + ', ' + i.family + ')</option>'
    ).join('');
    const opts = [...sel.options].map(o => o.value);
    if (opts.includes(s.bind_ip)) { sel.value = s.bind_ip; $('bindCustom').value = ''; }
    else { sel.value = '0.0.0.0'; $('bindCustom').value = s.bind_ip; }
    $('portInput').value = s.port;
    $('newUser').value = ''; $('newPass').value = ''; $('newPass2').value = '';
    $('portStatus').textContent = '';
    $('allowDelete').checked = !!s.allow_delete;
    const lsel = $('langSelect');
    if (lsel && s.language) lsel.value = s.language;
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}
function randomPort(){
  const min = 40000, max = 60000;
  $('portInput').value = Math.floor(Math.random() * (max - min)) + min;
  $('portStatus').textContent = t("ui.port_random");
  $('portStatus').style.color = 'var(--muted)';
}
async function checkPort(){
  const port = parseInt($('portInput').value, 10);
  if (!port) { $('portStatus').textContent = t("ui.port_enter"); return; }
  $('portStatus').innerHTML = '<span class="spin"></span> ' + esc(t("ui.checking"));
  try {
    const r = await apiRaw('/api/settings/check_port', {
      method: 'POST', headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({port})
    });
    $('portStatus').textContent = (r.ok ? '✓ ' : '✗ ') + r.reason;
    $('portStatus').style.color = r.ok ? 'var(--ok)' : 'var(--warn)';
  } catch(e){ $('portStatus').textContent = t("ui.error", {err: e.message}); $('portStatus').style.color = 'var(--warn)'; }
}
async function saveSettings(){
  const custom = $('bindCustom').value.trim();
  const bind_ip = custom || $('bindSelect').value;
  const port = parseInt($('portInput').value, 10);
  const username = $('newUser').value.trim();
  const pw1 = $('newPass').value; const pw2 = $('newPass2').value;
  const allow_delete = $('allowDelete').checked;
  const language = $('langSelect') ? $('langSelect').value : (I18N.lang || 'en');
  if (pw1 !== pw2) { toast(t("ui.password_mismatch")); return; }
  if (!port) { toast(t("ui.no_port")); return; }
  try {
    const r = await apiRaw('/api/settings', {
      method: 'POST', headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({bind_ip, port, username, password: pw1, allow_delete, language})
    });
    toast(r.message || t("ui.saved"));
    if (r.new_url) $('currentBind').textContent = t("ui.config_saved", {url: r.new_url});
  } catch(e){ toast(t("ui.error", {err: e.message})); }
}

(async function init(){
  applyI18n();
  try { await loadOverlays(); } catch(e){ console.error('loadOverlays', e); }
  try { loadDash(); } catch(e){ console.error('loadDash', e); }
  try { toggleAuthFields(); } catch(e){ console.error('toggleAuthFields', e); }
  try { updateSelInfo(); } catch(e){ console.error('updateSelInfo', e); }
})();
</script>
</body>
</html>
"""


# =============================================================
# MAIN
# =============================================================
def print_banner(cfg: dict) -> None:
    bind = cfg["bind_ip"]
    port = cfg["port"]
    display_ip = get_lan_ip() if bind in ("0.0.0.0", "::") else bind
    bar = "=" * 62
    print(f"\n{bar}")
    print(f"  {t('banner.running', app=APP_NAME)}")
    print(f"{bar}")
    print(f"  {t('banner.bind')}     {bind}:{port}")
    print(f"  {t('banner.local')}    http://127.0.0.1:{port}")
    if display_ip != "127.0.0.1":
        print(f"  {t('banner.network')} http://{display_ip}:{port}")
    print()
    print(f"  {t('banner.user')} {cfg['username']}")
    print(f"  {t('banner.password')}")
    print(f"  {t('banner.ssh_targets')} {len(cfg.get('ssh_targets', []))}")
    print(f"  {t('banner.disk_display', gb=DISK_MIN_AUTO_BYTES // (1024**3))}")
    print()
    print(f"  {t('banner.lan_only')}")
    print(f"  {t('banner.quit')}")
    print(f"{bar}\n")


def main():
    global _ACTIVE_LANG

    parser = argparse.ArgumentParser(description=APP_NAME)
    parser.add_argument("--reset", action="store_true",
                        help="Konfiguration (IP/Port/Login) neu einrichten")
    parser.add_argument("--config", type=str, default=None,
                        help="Pfad zur config.json")
    parser.add_argument("--lang", type=str, default=None, choices=list(SUPPORTED_LANGS),
                        help="Sprache für den Erststart (de/en/ru/ja/ko/zh)")
    args = parser.parse_args()

    # Sprache vor allem anderen festlegen (wird für t() gebraucht)
    if args.lang and args.lang in SUPPORTED_LANGS:
        _ACTIVE_LANG = args.lang
    else:
        # Aus Umgebungsvariable oder Default
        env_lang = os.getenv("PC_OVERLAY_LANG")
        if env_lang in SUPPORTED_LANGS:
            _ACTIVE_LANG = env_lang

    if args.config:
        config_path = Path(args.config).expanduser().resolve()
    else:
        config_path = Path(__file__).resolve().parent / "config.json"

    if args.reset and config_path.exists():
        config_path.unlink()
        print(f"{t('banner.config_deleted')} {config_path}")

    if not config_path.exists():
        cfg, config_path = interactive_setup(config_path)
    else:
        cfg = load_config(config_path)
        # Sprache aus Config übernehmen (falls per --lang nicht überschrieben)
        if args.lang and args.lang in SUPPORTED_LANGS:
            cfg["language"] = args.lang
        _ACTIVE_LANG = cfg.get("language") or DEFAULT_LANG
        if _ACTIVE_LANG not in SUPPORTED_LANGS:
            _ACTIVE_LANG = DEFAULT_LANG
            cfg["language"] = DEFAULT_LANG
        save_config(config_path, cfg)

    app = build_app(cfg, config_path)
    print_banner(cfg)

    try:
        uvicorn.run(
            app,
            host=cfg["bind_ip"],
            port=cfg["port"],
            log_level="warning",
            access_log=False,
        )
    except KeyboardInterrupt:
        print(f"\n{t('banner.stopped')}")
    except OSError as e:
        print(f"\n{t('banner.bind_error', ip=cfg['bind_ip'], port=cfg['port'])}")
        print(f"  {e}")
        sys.exit(1)


if __name__ == "__main__":
    main()
