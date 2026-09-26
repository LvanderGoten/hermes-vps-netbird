# Hermes Agent auf einem Hetzner-VPS – nur über NetBird erreichbar

Begleit-Repo zum Video. Hier liegen nur die Dateien, die man sonst aus Dokumentationen abtippen
müsste. Keine Schlüssel, keine echten IP-Adressen, keine persönliche Konfiguration.

Ergebnis: Hermes Agent, Papra und Vikunja laufen auf einem kleinen VPS. SSH, das Hermes-Dashboard
und beide Apps sind nur über NetBird erreichbar; die Hetzner Firewall hat null eingehende Regeln.

## Reihenfolge

1. **Hetzner Firewall** mit einer einzigen Regel anlegen: TCP 22 nur von deiner aktuellen IP (/32).
2. **Server bestellen** (z. B. CAX11, Ubuntu 26.04), öffentliche SSH-Keys auswählen, Firewall anhängen.
3. **Erster Login** als root, Updates, Admin-Benutzer:
   ```bash
   apt update && apt full-upgrade -y
   adduser --gecos "" ops
   usermod -aG sudo ops
   rsync --archive --chown=ops:ops ~/.ssh /home/ops
   ```
4. **SSH absichern** (zweites Fenster, als `ops`): `ssh/10-hardening.conf` nach
   `/etc/ssh/sshd_config.d/` kopieren, dann
   ```bash
   sudo sshd -t && sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|allowusers|authenticationmethods'
   sudo systemctl reload ssh
   ```
5. **NetBird**: im Dashboard einen Setup Key (einmalig, 1 Tag, Auto-Gruppe `agents`)
   erstellen, dann auf dem Server
   ```bash
   curl -fsSL https://pkgs.netbird.io/install.sh | sh
   read -rs NB_SETUP_KEY
   sudo netbird up --setup-key "$NB_SETUP_KEY"
   unset NB_SETUP_KEY
   netbird status
   ```
   Gruppen `admins` (Laptop) und `agents` (Server). Zugriffsregeln (eine Richtung,
   `admins` → `agents`): TCP 22 und TCP 3456, 1221, 9119.
   Unter **DNS → Zones** eine private Zone `agent.internal` für beide Gruppen anlegen, mit den
   Records `hermes`, `vikunja` und `papra` auf den Server. Danach genügen Namen statt IPs:
   `ssh ops@hermes.agent.internal`.
   Alle anderen Regeln prüfen, besonders eine breite Standardregel.
6. **Privat testen, dann schließen**: neue SSH-Verbindung über den NetBird-Namen. Erst danach die
   Firewall-Regel löschen. Gegenprobe: `nc -vz -w 5 <öffentliche-IP> 22` läuft in einen Timeout.
   Läuft auf deinem Rechner zusätzlich NordVPN, braucht die NetBird-IP des Servers dort eine
   Ausnahme: `nordvpn allowlist add subnet <netbird-ip>/32`.
7. **Docker** aus der offiziellen Paketquelle (docs.docker.com/engine/install/ubuntu), dann
   `sysctl/60-netbird-bind.conf` nach `/etc/sysctl.d/` und `sudo sysctl --system`.
8. **Vikunja und Papra**: `vikunja/compose.yaml` nach `/opt/vikunja`, `papra/compose.yaml` nach
   `/opt/papra`. Die `.env`-Dateien ohne sichtbare Geheimnisse anlegen, z. B.
   ```bash
   sudo tee /opt/vikunja/.env >/dev/null <<EOF
   NETBIRD_IP=$(netbird status --ipv4)
   PUBLIC_URL=http://vikunja.agent.internal:3456/
   VIKUNJA_JWT_SECRET=$(openssl rand -hex 32)
   EOF
   ```
   (für Papra `PUBLIC_URL=http://papra.agent.internal:1221` und `PAPRA_AUTH_SECRET=$(openssl rand -hex 48)`), dann `sudo docker compose up -d`.
   Konten anlegen, danach `VIKUNJA_REGISTRATION=false` bzw. `PAPRA_REGISTRATION=false` in die
   `.env` und `sudo docker compose up -d`.
9. **Hermes Agent** als eigener Benutzer ohne sudo, gepinnt auf v0.21.5:
   ```bash
   sudo apt install -y build-essential xz-utils ripgrep
   sudo adduser --disabled-password --comment "" hermes
   sudo -iu hermes
   curl -fsSLo install.sh https://raw.githubusercontent.com/NousResearch/hermes-agent/v2026.9.24/scripts/install.sh
   bash install.sh --commit f97608f178d1ffeca59860195ab7da295f7c8e5f --skip-setup
   source ~/.bashrc
   hermes model        # OpenRouter, Key mit Ausgabenlimit, Modell z-ai/glm-5.3-flash
   ```
   Der Installer bietet an, ffmpeg und weitere Build-Tools per sudo nachzuinstallieren: beides mit
   `n` ablehnen, der Benutzer `hermes` hat bewusst kein sudo (`build-essential` kam vorher über
   `ops`). Die Frage von `npx`, ob es Playwright installieren darf, mit `y` bestätigen.
10. **Skills installieren und Anbieter festlegen**: die Skills vom Laptop auf den Server kopieren und
    dem Benutzer `hermes` übergeben:
    ```bash
    git clone https://github.com/LvanderGoten/hermes-vps-netbird.git
    scp -r hermes-vps-netbird/skills ops@hermes.agent.internal:
    ssh ops@hermes.agent.internal
    sudo cp -r skills/papra skills/vikunja /home/hermes/.hermes/skills/ && sudo chown -R hermes:hermes /home/hermes/.hermes/skills
    sudo -iu hermes
    ```
    OpenRouter verteilt ein Modell sonst nach Preis auf viele Anbieter, auch auf stark quantisierte
    Varianten. Deshalb nur den Modellanbieter selbst zulassen:
    ```bash
    hermes config set provider_routing.only '["z-ai"]' --force
    ```
    (`--force` unterdrückt nur einen Hinweis; Hermes liest den Schlüssel trotzdem.)
    Beim ersten Laden fragt Hermes die Tokens verdeckt ab. Vor dem ersten API-Aufruf mit einem Token
    fragt Hermes außerdem nach einer Freigabe („Dangerous Command“): „Allow for this session“.
11. **Dashboard** nur auf der NetBird-Adresse:
    ```bash
    hermes config set dashboard.public_url http://hermes.agent.internal:9119
    hermes dashboard --host $(netbird status --ipv4) --no-open   # Benutzername und Passwort festlegen
    ```
    Dauerhaft als Dienst: `hermes/hermes-dashboard.service` nach `/etc/systemd/system/`, dazu
    `/etc/hermes-dashboard.env` mit `NETBIRD_IP=…`, dann `sudo systemctl enable --now hermes-dashboard`.
12. **Gegenprobe und Neustart**: alle vier Ports von außen testen, neu starten, erneut testen.

## Grenzen

Wer dein NetBird-Konto übernimmt, steht im selben Netz: Mehrfaktor-Anmeldung einschalten.
Updates und Backups bleiben deine Aufgabe. Was Hermes darf, bestimmen die Rechte seiner Tokens
und sein Linux-Benutzer, nicht das Netz.
