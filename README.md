# Ansible-Wartung für Proxmox VE

Playbooks zur Wartung von LXC-Containern und VMs auf Proxmox VE:
Snapshot anlegen, patchen, bei Bedarf neu starten, aufräumen und den Zustand
berichten.

Läuft mit reinem **ansible-core** — es müssen keine Collections installiert
werden. Die Proxmox-Ebene wird über `pvesh`, `pct` und `qm` direkt auf dem
PVE-Host angesprochen, es braucht also weder API-Token noch `proxmoxer` auf
dem Controller.

## Von Null zum ersten Lauf

Diese Anleitung führt von einem frisch geklonten Repo bis zum ersten
ausgeführten Playbook. Schritt 2 ist der einzige, den du von Hand machen
musst — danach übernimmt Ansible.

### 1. Controller vorbereiten

Der Controller ist der Rechner, von dem aus du Ansible startest. Er braucht
**Linux** und **ansible-core ab 2.14** (die Rollen verwenden
`ansible.builtin.systemd_service`, das es erst ab dieser Fassung gibt).

```bash
# Debian/Ubuntu
sudo apt install ansible-core

# oder über pip, wenn die Distribution eine zu alte Fassung mitbringt
pip install "ansible-core>=2.14"

git clone https://github.com/Ighwaz/ansible && cd ansible
ansible --version        # muss 2.14 oder neuer zeigen
```

Collections sind **nicht** nötig. Für Linting und die lokale Prüfung optional:
`pip install -r requirements-dev.txt`.

> **Windows geht nicht als Controller.** Ansible unterstützt Windows nur als
> *Ziel*, nicht als Steuerrechner. Eine Installation per pip unter Windows
> lässt sich zwar durchführen, bricht aber beim ersten Aufruf ab:
>
> ```
> File "...\ansible\cli\__init__.py", line 46, in check_blocking_io
>     if not os.get_blocking(fd):
> OSError: [WinError 1] Unzulässige Funktion
> ```
>
> `check_blocking_io()` läuft beim Import der CLI, noch vor allem anderen, und
> `os.get_blocking()` gibt es nur unter Unix. Das ist kein Fehler in deiner
> Installation und nicht reparierbar.

#### Unter Windows: WSL oder ein Linux-Gast

**Der schnelle Weg — WSL2.** In PowerShell:

```powershell
wsl --install -d Ubuntu
```

Danach in der Ubuntu-Shell:

```bash
sudo apt update && sudo apt install -y ansible-core git
git clone https://github.com/Ighwaz/ansible ~/ansible
cd ~/ansible && ansible --version
```

> **Das Repo muss im WSL-Dateisystem liegen, nicht unter `/mnt/c`.** WSL
> mountet Windows-Laufwerke welt-schreibbar (777), und Ansible **ignoriert
> seine `ansible.cfg` in welt-schreibbaren Verzeichnissen** — mit einer
> Warnung, die man leicht überliest:
>
> ```
> [WARNING]: Ansible is being run in a world writable directory,
> ignoring it as an ansible.cfg source.
> ```
>
> Damit fielen still alle Einstellungen dieses Repos weg: der Inventory-Pfad,
> `become = True`, `roles_path`. Die Playbooks liefen dann scheinbar, aber
> gegen das falsche Inventory und ohne root-Rechte. Deshalb `~/ansible` statt
> `/mnt/c/Users/...`.
>
> Mit IntelliJ oder VS Code kommst du trotzdem bequem an die Dateien:
> `\\wsl$\Ubuntu\home\<benutzer>\ansible` öffnen — bearbeitet wird unter
> Windows, ausgeführt in WSL.

**Der aufgeräumte Weg — ein LXC auf dem Proxmox selbst.** Ein kleiner
Debian-Container (1 vCPU, 512 MB) als Ansible-Controller: immer erreichbar,
kein Dateisystem-Grenzfall, und er steht schon im selben Netz wie alles, was
er verwaltet. Dann wird dieser Container Teil deines Inventories wie jeder
andere Gast auch.

### 2. Zugang auf den Zielhosts schaffen (einmalig, von Hand)

Hier liegt das Henne-Ei-Problem: Ansible kann Zugänge einrichten, braucht dafür
aber selbst schon einen. Dieser eine Schritt geht also manuell.

Zuerst auf dem Controller einen Schlüssel erzeugen, falls noch keiner da ist:

```bash
ssh-keygen -t ed25519 -C "ansible@controller" -f ~/.ssh/id_ansible
cat ~/.ssh/id_ansible.pub          # diesen Text brauchst du gleich
```

Dann auf **jedem** PVE-Host und **jedem** Gast als root ausführen — bei
LXC-Containern am einfachsten über `pct enter <vmid>` auf dem PVE-Host, bei
VMs über die Konsole im Webinterface:

```bash
# Benutzer anlegen (ohne Passwort-Anmeldung)
adduser --disabled-password --gecos "" ansible

# Öffentlichen Schlüssel hinterlegen
install -d -m 700 -o ansible -g ansible /home/ansible/.ssh
echo 'ssh-ed25519 AAAA...HIER_DEINEN_SCHLUESSEL... ansible@controller' \
  > /home/ansible/.ssh/authorized_keys
chown ansible:ansible /home/ansible/.ssh/authorized_keys
chmod 600 /home/ansible/.ssh/authorized_keys

# sudo ohne Passwort — zwingend, siehe Kasten unten
echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible
chmod 440 /etc/sudoers.d/ansible

# In sehr schlanken Containern fehlt beides gelegentlich
apt install -y python3 sudo
```

> **Warum sudo ohne Passwort?** `ansible.cfg` setzt `become = True` — jeder
> Aufruf wird zu root. Fragt sudo nach einem Passwort, scheitert schon der
> erste `ping`. Alternativ kannst du in `inventory/group_vars/all.yml`
> `ansible_user: root` setzen und dich direkt als root anmelden; dann entfällt
> die sudoers-Zeile.

**Host-Schlüssel vorab bestätigen.** `ansible.cfg` setzt
`host_key_checking = True`. Beim allerersten Kontakt kennt der Controller die
Host-Schlüssel noch nicht und bricht ab. Entweder einmal je Host von Hand
verbinden (`ssh ansible@192.0.2.21`) oder alle auf einmal aufnehmen:

```bash
ssh-keyscan -H 192.0.2.10 192.0.2.21 192.0.2.22 >> ~/.ssh/known_hosts
```

### 3. Inventory ausfüllen

`inventory/hosts.yml` enthält Platzhalter (`192.0.2.x` ist der reservierte
Beispiel-Adressbereich) — die müssen alle raus. Welche Gäste es gibt, verrät
der PVE-Host:

```bash
ssh root@<dein-pve-host> "pvesh get /cluster/resources --type vm"
```

Daraus wird das Inventory. Trage die Container unter `lxc`, die VMs unter
`vms` und die PVE-Hosts unter `proxmox` ein:

```yaml
    proxmox:
      hosts:
        pve01:
          ansible_host: 10.0.0.10

    lxc:
      hosts:
        ct-nginx:
          ansible_host: 10.0.0.21

    vms:
      hosts:
        vm-build:
          ansible_host: 10.0.0.31
```

**Eine `vmid` musst du nicht pflegen**, solange der Inventory-Name dem Namen in
Proxmox entspricht — Ansible löst vmid, Node und Typ selbst auf. Weicht der
Name ab, genügt `pve_vmid: 123` beim betreffenden Host.

Die Gruppen `docker_hosts` und `monitoring` sind nur nötig, wenn du Docker
bzw. Icinga verwendest. Nicht benötigte Gruppen einfach leeren.

### 4. Zugangsdaten eintragen

In `inventory/group_vars/all.yml`:

```yaml
ansible_user: ansible
ansible_ssh_private_key_file: ~/.ssh/id_ansible
```

### 5. Verbindung prüfen

```bash
ansible all -m ping
```

Das prüft in einem Zug SSH-Anmeldung **und** sudo, weil `become = True` gilt.
Erwartet wird `SUCCESS` für jeden Host. Häufige Fehlermeldungen:

| Meldung | Ursache |
|---|---|
| `Permission denied (publickey)` | Schlüssel nicht hinterlegt oder falscher `ansible_user` |
| `Host key verification failed` | Schritt 2, letzter Absatz — Host-Schlüssel fehlt in `known_hosts` |
| `sudo: a password is required` | sudoers-Zeile fehlt oder ohne `NOPASSWD` |
| `/usr/bin/python3: not found` | im Gast fehlt `python3` |
| `Failed to connect ... Connection refused` | falsche IP, oder sshd läuft nicht |

### 6. Der erste Lauf

Jetzt in dieser Reihenfolge — die ersten beiden ändern **nichts**:

```bash
# 1. Prüft, ob alle Voraussetzungen wirklich erfüllt sind
ansible-playbook playbooks/preflight.yml

# 2. Liest den Zustand aus und schreibt einen Report nach reports/
ansible-playbook playbooks/healthcheck.yml

# 3. Erst jetzt etwas Veränderndes — zuerst als Probelauf,
#    und nur gegen einen unkritischen Host
ansible-playbook playbooks/maintenance.yml --limit ct-nginx --check
ansible-playbook playbooks/maintenance.yml --limit ct-nginx
```

`preflight.yml` ist der eigentliche Einstiegspunkt: Es prüft, ob `pvesh`,
`pct` und `qm` erreichbar sind, ob die Cluster-Antwort die erwarteten Felder
enthält, ob `become` tatsächlich als root ankommt und ob `python3-apt` für
`--check`-Läufe vorhanden ist. Meldet es Befunde, stimmen die Annahmen dieses
Repos noch nicht mit deiner Umgebung überein — dann dort anfangen.

### 7. Erst danach: Schlüssel, Docker, Monitoring

Die übrigen Bereiche bauen auf einem funktionierenden Zugang auf und haben
eigene Abschnitte weiter unten. Sinnvolle Reihenfolge:

1. `ssh-keys.yml` — deine persönlichen Schlüssel ausrollen (**ohne** Härtung,
   siehe Abschnitt „SSH-Schlüssel")
2. `pve-node-setup.yml` — falls ein Node noch grundeingerichtet werden muss
3. `docker-setup.yml` — nur für Hosts in `docker_hosts`
4. `icinga-setup.yml` — braucht einen Monitoring-Host mit Docker

## Die Playbooks

| Playbook | Zweck | Verändert etwas? |
|---|---|---|
| `playbooks/preflight.yml` | Voraussetzungen prüfen, bevor irgendetwas läuft | nein |
| `playbooks/healthcheck.yml` | Zustand aller Gäste erfassen, Markdown-Report unter `reports/` | nein |
| `playbooks/maintenance.yml` | Komplette Wartung: Snapshot → Updates → Neustart → Aufräumen | ja |
| `playbooks/update.yml` | Nur Snapshot + Updates + Neustart | ja |
| `playbooks/cleanup.yml` | Nur Housekeeping (Paketreste, Cache, Journal) | ja |
| `playbooks/snapshot.yml` | Nur Snapshots anlegen, sonst nichts | ja (nur Snapshot) |
| `playbooks/snapshot-prune.yml` | Alte Ansible-Snapshots aufräumen | ja (nur Snapshots) |
| `playbooks/pve-node-setup.yml` | Neuen Proxmox-Node grundeinrichten | ja |
| `playbooks/pve-host-update.yml` | Die PVE-Hosts selbst aktualisieren | ja |
| `playbooks/docker-setup.yml` | Docker installieren (rootful oder rootless) | ja |
| `playbooks/docker-update.yml` | Docker selbst aktualisieren, mit Container-Kontrolle | ja |
| `playbooks/icinga-setup.yml` | Icinga2-Stack aufsetzen und Gäste vorbereiten | ja |
| `playbooks/icinga-config.yml` | Prüfungen aus dem Inventory neu erzeugen | ja |
| `playbooks/ssh-keys.yml` | SSH-Schlüssel ausrollen, optional sshd härten | ja |

### Preflight im Detail

Der Einstiegspunkt aus Schritt 6 oben, hier vollständig beschrieben.

```bash
ansible-playbook playbooks/preflight.yml
```

Prüft in einem Rutsch, ob alles steht, was die übrigen Playbooks voraussetzen,
und bricht dabei nicht beim ersten Problem ab, sondern sammelt alle Befunde:

- **PVE-Hosts:** sind `pvesh`, `pct`, `qm` und `pveversion` aufrufbar, antwortet
  `pvesh get /cluster/resources`, und enthält die Antwort die Felder, auf die
  sich `pve_facts` stützt (`vmid`, `node`, `type`, `name`, `status`)?
- **Gäste:** Debian/Ubuntu, kommt `become` wirklich als root an, ist
  `python3-apt` für `--check` vorhanden, und lässt sich der Gast einem
  Proxmox-Gast zuordnen?

Der Exit-Code ist aussagekräftig: ungleich 0, sobald ein Host Befunde hat.
Nur berichten statt scheitern geht mit `-e preflight_fail_on_problems=false`.

Der Feldcheck ist bewusst streng, weil die Rollen gegen eine bestimmte Form
der Cluster-Antwort entwickelt wurden. Weicht deine PVE-Version ab, fällt das
hier auf und nicht mitten in einem Wartungslauf.

### Typische Abläufe

```bash
# Erst mal nur schauen, wie es um alles steht
ansible-playbook playbooks/healthcheck.yml

# Einen einzelnen Container warten
ansible-playbook playbooks/maintenance.yml --limit ct-nginx

# Alle Container warten, VMs außen vor lassen
ansible-playbook playbooks/maintenance.yml --limit lxc

# Alles warten, dabei vier Hosts gleichzeitig
ansible-playbook playbooks/maintenance.yml -e maint_serial=4

# Updates ohne Snapshot (z. B. wenn das Storage keine Snapshots kann)
ansible-playbook playbooks/update.yml -e pve_snapshot_enabled=false

# Die Proxmox-Hosts selbst patchen
ansible-playbook playbooks/pve-host-update.yml
```

## Was die Playbooks für dich mitdenken

**Snapshots als Rückfallebene.** Vor jedem Update entsteht ein Snapshot
`ansible-<zeitstempel>`. Schlägt der Snapshot fehl, wird der betreffende Host
**nicht** gepatcht — ein Update ohne Rückfallebene passiert nicht versehentlich
(abschaltbar über `pve_snapshot_fail_on_error`).

**Aufräumen ohne Kollateralschaden.** Beim Prune werden ausschließlich
Snapshots angefasst, deren Name mit `ansible-` beginnt. Manuell angelegte
Snapshots bleiben unberührt, egal wie alt sie sind.

**Reboots dort, wo sie hingehören.** LXC-Container werden über den PVE-Host
mit `pct reboot` neu gestartet statt von innen — das fährt den Container-
Lifecycle sauber durch. VMs starten sich selbst neu.

**Kernel-Updates nur bei VMs.** LXC-Container nutzen den Kernel des Hosts, ein
Kernel-Vergleich im Container wäre bedeutungslos. Für Container zählen daher
nur `/var/run/reboot-required` und `needrestart`.

**Der Hypervisor startet nie automatisch neu.** Ein Reboot des PVE-Hosts
reißt alle Gäste mit. `pve-host-update.yml` meldet einen anstehenden Neustart
nur — einplanen musst du ihn selbst.

**Ein defekter Host stoppt nicht die Wartung aller anderen.** Die Playbooks
setzen `max_fail_percentage: 100`; ohne das würde bei `serial: 1` ein einziger
fehlgeschlagener Host das gesamte Play beenden.

**Host für Host statt alles auf einmal.** Standard ist `maint_serial: 1`.
Fällt ein Dienst beim Neustart aus, betrifft das immer nur einen Gast.

## Neuen Proxmox-Node einrichten

```bash
ansible-playbook playbooks/pve-node-setup.yml --check --limit pve02
ansible-playbook playbooks/pve-node-setup.yml --limit pve02
```

Deckt ab: Paketquellen (Enterprise aus, no-subscription an), Zeitzone und
Zeitabgleich, Locale, Basiswerkzeuge, Vorgaben für Sicherungen und einen
lesenden Abgleich des erwarteten Storages.

### Was die Rolle bewusst nicht tut

Zwei Dinge bleiben manuell, weil ein Fehlgriff dort teuer ist:

- **Kein Cluster-Beitritt.** `pvecm add` auf dem falschen Node oder zur
  falschen Zeit beschädigt einen bestehenden Cluster.
- **Kein Anlegen von Storage.** Das Erzeugen eines ZFS-Pools löscht die
  angegebenen Datenträger. Die Rolle prüft nur, ob die unter
  `pve_setup_expected_storages` erwarteten Storages vorhanden sind, und meldet
  Abweichungen. Sie unterscheidet dabei zwischen „fehlt" und „nicht
  ermittelbar" — scheitert `pvesm status`, ist der Zustand unbekannt, nicht
  leer.

Die Repo-Logik wird aus `pve_host_update` wiederverwendet statt verdoppelt:
die Unterscheidung zwischen dem `.list`-Format (PVE 8) und deb822 (PVE 9)
wird nur an einer Stelle gepflegt.

### Abo-Hinweis im Webinterface

`pve_setup_hide_subscription_notice` blendet den Hinweis nach dem Anmelden
aus. Das verändert eine mitgelieferte Datei von Proxmox — zwei Dinge dazu:

- Ein Update von `proxmox-widget-toolkit` setzt die Änderung zurück; ein
  erneuter Lauf stellt sie wieder her. Eine Sicherungskopie bleibt jeweils
  daneben liegen.
- Ersetzt wird die **gesamte** Bedingung, nicht nur ein Ausschnitt. Der
  verbreitete Einzeiler, der bloß `data.status.toLowerCase() !== 'active'`
  ersetzt, hinterlässt `res.false` — das wirkt zwar richtig, aber nur zufällig,
  weil ein reserviertes Wort als Eigenschaftsname erlaubt ist und `undefined`
  als falsch gilt. Hier steht danach schlicht `if (false /* ... */)`.

Erkennt die Rolle den Aufbau der Datei nicht wieder — etwa nach einem
Proxmox-Update —, meldet sie das und lässt die Datei unangetastet, statt zu
raten.

### Vorgaben für Sicherungen

`/etc/vzdump.conf` bekommt Modus, Komprimierung, optional eine
Bandbreitenbegrenzung und eine gestaffelte Aufbewahrung über `prune-backups`.
Leere Werte werden weggelassen, dann gilt weiterhin die Vorgabe von Proxmox.
Ein Sicherungsauftrag mit eigenen Einstellungen sticht diese Vorgaben.

## Docker

Hosts mit Docker stehen im Inventory in der Gruppe `docker_hosts`. Ob ein Host
klassisch als root oder rootless fährt, entscheidet `docker_rootless` — je Host
umschaltbar:

```yaml
docker_hosts:
  hosts:
    vm-build:
      docker_rootless: true
    vm-home-assistant:      # ohne Angabe: rootful
```

```bash
ansible-playbook playbooks/docker-setup.yml
ansible-playbook playbooks/docker-update.yml --limit vm-build
```

### Warum Docker-Updates ein eigenes Playbook haben

`guest_update` fährt ein `apt upgrade` und würde Docker dabei einfach
mitnehmen — mitten im Wartungslauf, ohne Rücksicht darauf, was gerade an
Containern läuft. Deshalb setzen beide Docker-Rollen die Pakete auf `hold`
(`docker_update_hold: true`), womit `apt upgrade` sie in Ruhe lässt.

`docker-update.yml` hebt den Hold auf, aktualisiert, setzt ihn wieder — und
erfasst dabei **vor und nach** dem Update, welche Container laufen. Kommt einer
nicht zurück, bricht das Playbook mit dessen Namen ab. War der Daemon schon
vorher nicht erreichbar, wird das ausdrücklich als solches gemeldet statt dem
Update angelastet.

### Rootless: was du wissen solltest

- Der Daemon läuft in der systemd-User-Instanz von `docker_rootless_user`.
  Damit er einen Neustart überlebt, wird für den Benutzer **Lingering**
  aktiviert; der Socket liegt unter `/run/user/<uid>/docker.sock`.
- Der systemweite Daemon wird abgeschaltet
  (`docker_rootless_disable_system_daemon`). Beide parallel zu betreiben ist
  möglich, aber eine verlässliche Quelle von Verwirrung darüber, gegen welchen
  Daemon ein `docker`-Aufruf gerade läuft.
- `live-restore` wird im rootless-Betrieb nicht gesetzt — der Daemon
  unterstützt es dort nicht.
- Fehlen dem Benutzer Einträge in `/etc/subuid` und `/etc/subgid`, bricht die
  Rolle mit dem passenden `usermod`-Aufruf ab, statt Bereiche selbst zu
  vergeben. Eine falsch gewählte Spanne kollidiert mit anderen Benutzern und
  ist hinterher mühsam zu entwirren.
- **In LXC-Containern** verweigert die Rolle rootless-Betrieb. Das braucht
  `nesting=1` und `keyctl=1` in der Container-Config auf dem PVE-Host, was
  diese Rolle nicht einrichtet. Bewusst übergehen mit
  `docker_allow_rootless_in_lxc: true`.

### Zur Gruppe `docker`

`docker_users` ist bewusst leer. Mitgliedschaft in der Gruppe `docker` ist
praktisch gleichbedeutend mit root-Rechten auf dem Host: wer den Socket
erreicht, kann einen privilegierten Container starten und damit das
Wirtssystem übernehmen.

## Monitoring mit Icinga2

Der Icinga-Stack läuft als Docker-Verbund auf dem Host der Gruppe
`monitoring` — dieser Host muss also auch in `docker_hosts` stehen. Geprüft
wird **agentless über SSH**: kein Agent auf den Gästen, keine Zertifikate,
kein PKI-Aufbau.

```bash
ansible-playbook playbooks/docker-setup.yml --limit vm-monitoring   # zuerst
ansible-playbook playbooks/icinga-setup.yml                         # dann
```

`icinga-setup.yml` erledigt drei Dinge nacheinander: es startet den Stack und
erzeugt dabei ein SSH-Schlüsselpaar, richtet auf jedem Gast den
unprivilegierten Zugang `icinga` samt Monitoring-Plugins ein und hinterlegt
dort den öffentlichen Schlüssel, und erzeugt zuletzt die Host- und
Service-Definitionen aus dem Inventory.

Im Alltag genügt danach `icinga-config.yml` — etwa wenn ein Gast dazukommt
oder ein Schwellwert sich ändert. Ein **neuer** Gast braucht allerdings
einmalig `icinga-setup.yml`, weil dort sein Monitoring-Zugang entsteht.

### Der Stack

| Dienst | Aufgabe |
|---|---|
| `icinga2` | der Monitoring-Kern, führt die Prüfungen aus |
| `redis` | Zwischenspeicher, über den icinga2 Zustände abliefert |
| `icingadb` | überträgt die Zustände von Redis in die Datenbank |
| `mariadb` | Datenbank für Icinga DB und Icinga Web 2 |
| `icingaweb2` | Weboberfläche, per Vorgabe auf Port 8080 |

Die Image-Tags sind gepinnt statt `latest`: ein unbemerkter Sprung auf eine
neue Major-Version ist ausgerechnet beim Monitoring das Letzte, was man will.

Passwörter werden beim ersten Lauf erzeugt und in `/opt/icinga/.env` abgelegt
(nur für root lesbar). Sie stehen bewusst nicht im Repository.

### Welche Prüfungen entstehen

Je Gast, gesteuert über `icinga_services_enabled`:

| Prüfung | Herkunft | Anmerkung |
|---|---|---|
| `ping`, `ssh` | vom Master direkt | ohne SSH-Anmeldung |
| `disk` | `check_disk` | Schwellwerte in **freiem** Platz |
| `load` | `check_load -r` | umgerechnet auf CPU-Kerne |
| `swap` | `check_swap` | **nicht für Container** — siehe unten |
| `procs` | `check_procs` | |
| `apt` | `check_apt` | ausstehende Updates, Intervall 1 h |
| `memory` | eigenes Skript | rechnet gegen `MemAvailable` |
| `systemd` | eigenes Skript | fehlgeschlagene Units |

Zwei Prüfungen bringt das Repo selbst mit, weil `monitoring-plugins` sie nicht
enthält. Beide liegen unter `roles/icinga_remote/files/`.

**Swap läuft nicht in Containern.** Ein LXC sieht den Swap des Wirts. Der Wert
sagt dort nichts über den Container aus und würde bei Swap-Druck auf dem Host
die gesamte Flotte gleichzeitig rot färben — dieselbe Unterscheidung, die auch
die Wartungsrollen beim Kernel treffen.

### Sicherheit des Monitoring-Zugangs

Der Benutzer `icinga` auf den Gästen ist unprivilegiert und hat kein sudo.
Sein Schlüssel in `authorized_keys` ist eingeschränkt auf die Adresse des
Monitoring-Hosts, ohne Weiterleitungen und ohne Terminal. Auf einen einzelnen
erzwungenen Befehl lässt sich das nicht reduzieren: `check_by_ssh` schickt die
jeweilige Prüfung als Kommando mit.

Host-Schlüssel werden mit `StrictHostKeyChecking=accept-new` beim ersten
Kontakt übernommen. Das schützt nicht gegen einen Angreifer, der schon beim
allerersten Kontakt dazwischensitzt — dagegen hülfe nur ein vorab befülltes
`known_hosts`.

### Mitgelieferte Beispielkonfiguration

Das icinga2-Image bringt eine eigene Beispielkonfiguration in `conf.d` mit,
deren `apply`-Regeln zusätzlich auf die von Ansible erzeugten Hosts greifen.
Konkret entsteht dadurch neben der Prüfung `ping` noch ein `ping4` je Host.
Doppelte Prüfungen stören nicht, kosten aber Platz in der Oberfläche — wer sie
loswerden will, entfernt die Beispieldateien im Volume `icinga2-data`.

## SSH-Schlüssel

Öffentliche Schlüssel stehen in `inventory/group_vars/all.yml`. Der Name je
Eintrag landet als Kommentar in der `authorized_keys` und macht später
eindeutig, welcher Schlüssel zu welchem Gerät gehört.

```yaml
ssh_keys_admin:
  - name: laptop
    key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... tofa@laptop"

# Der Schlüssel, mit dem Ansible selbst arbeitet
ssh_keys_automation:
  - name: ansible
    key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... ansible@controller"
```

**Nur öffentliche Schlüssel.** Private gehören niemals ins Repository.

```bash
ansible-playbook playbooks/ssh-keys.yml --check     # erst ansehen
ansible-playbook playbooks/ssh-keys.yml
```

Ausgerollt wird an den Ansible-Benutzer und an `root`; wer was bekommt, regelt
`ssh_keys_targets`.

### Widerruf

Die `authorized_keys` wird **vollständig** verwaltet (`ssh_keys_exclusive`).
Erst dadurch ist ein Widerruf möglich: Eintrag aus `ssh_keys_admin` entfernen,
Playbook laufen lassen — der Schlüssel ist flottenweit weg. Von Hand
hinzugefügte Schlüssel verschwinden damit allerdings ebenfalls.

### Schutz gegen das Aussperren

Genau hier geht es schief, und zwar endgültig. Drei Sicherungen greifen:

1. **Vor dem Schreiben** prüft die Rolle für den Benutzer, als der Ansible
   gerade verbunden ist, ob mindestens einer der bisherigen Schlüssel auch in
   der neuen Liste steht. Ist das nicht der Fall, bricht der Lauf ab, ohne die
   Datei anzufassen — sonst würde er den eigenen Zugang entfernen. Bewusst
   übergehen mit `-e ssh_keys_force=true`.
2. **Nach dem Schreiben** liest `ssh-keygen -l -f` die Datei so, wie sshd es
   täte, und gleicht die Anzahl erkannter Schlüssel mit der Konfiguration ab.
   Eine Datei, in der die Schlüssel versehentlich auskommentiert sind, sieht
   auf den ersten Blick richtig aus — gewährt aber niemandem Zugang.
3. **Der Ansible-Automatisierungsschlüssel** gehört in `ssh_keys_automation`.
   Fehlt er dort, entfernt ihn der exklusive Lauf; Sicherung 1 fängt das ab.

### Härtung von sshd

`ssh_keys_disable_password_auth: true` schaltet die Passwort-Anmeldung ab.
Standardmäßig aus: erst ausrollen und überprüfen, dann härten.

Vier Bedingungen müssen erfüllt sein, sonst bricht die Rolle ab, ohne etwas
zu ändern:

- Ansible ist **nicht** selbst per Passwort verbunden — sonst sperrt der Lauf
  die Automatisierung mit sich selbst aus
- sshd läuft überhaupt
- `sshd_config` bindet `/etc/ssh/sshd_config.d/` ein, die Drop-in-Datei wäre
  sonst wirkungslos und die Härtung nur scheinbar aktiv
- `sshd -t` akzeptiert die Konfiguration — die Prüfung läuft **vor** dem
  Schreiben, eine ungültige Datei entsteht gar nicht erst

Anschließend wird `reload` statt `restart` verwendet: bestehende Sitzungen
bleiben offen und damit als Rettungsanker erhalten. Danach setzt die Rolle die
Verbindung zurück und baut eine **frische** auf — nur die beweist, dass die
neue Konfiguration trägt.

Rückgängig machen heißt, eine Datei zu löschen:

```bash
rm /etc/ssh/sshd_config.d/60-ansible-hardening.conf && systemctl reload ssh
```

`PermitRootLogin` wird bewusst nicht angefasst. Ein falscher Wert dort nimmt
den root-Zugang, der bei kaputtem sudo die letzte Rückfallebene ist.

## Wichtige Variablen

Zu setzen in `inventory/group_vars/guests.yml`, pro Host im Inventory oder
per `-e` auf der Kommandozeile.

| Variable | Standard | Bedeutung |
|---|---|---|
| `maint_serial` | `1` | Wie viele Hosts gleichzeitig gewartet werden |
| `maint_reboot_allowed` | `true` | Darf der Host automatisch neu starten? |
| `pve_snapshot_enabled` | `true` | Snapshot vor dem Patchen |
| `pve_snapshot_keep` | `3` | Wie viele Ansible-Snapshots je Gast bleiben |
| `pve_snapshot_vmstate` | `false` | RAM-Zustand laufender VMs mitsichern |
| `guest_upgrade_type` | `safe` | `safe` = `apt upgrade`, `full` = `dist-upgrade` |
| `guest_hold_packages` | `[]` | Pakete, die nie aktualisiert werden |
| `cleanup_docker_prune` | `false` | `docker system prune` beim Aufräumen |
| `health_disk_warn_percent` | `85` | Ab wann ein Dateisystem im Report auffällt |

Einzelne Hosts lassen sich gezielt ausnehmen, z. B. eine Datenbank, die nur
im Wartungsfenster neu starten darf:

```yaml
lxc:
  hosts:
    ct-db:
      ansible_host: 192.0.2.22
      maint_reboot_allowed: false
      guest_hold_packages:
        - postgresql-16
```

## Rollback

Ist nach einem Update etwas kaputt, führt der Snapshot zurück. Auf dem
PVE-Host:

```bash
pct listsnapshot 105            # bzw. qm listsnapshot 201 für VMs
pct rollback 105 ansible-20260823T120000
```

## Dry-Run

Alle Playbooks laufen mit `--check`:

```bash
ansible-playbook playbooks/maintenance.yml --limit ct-nginx --check
```

Im Check-Modus entstehen **keine** Snapshots und es wird nichts gelöscht oder
neu gestartet. Eine Eigenheit des `apt`-Moduls: Für `--check` muss auf dem Gast
`python3-apt` installiert sein (bei normalen Läufen installiert Ansible es bei
Bedarf selbst nach). Auf Debian/Ubuntu ist das Paket üblicherweise vorhanden;
fehlt es, hilft `apt install python3-apt`.

## Der Health-Report

`healthcheck.yml` schreibt einen Markdown-Report nach `reports/` mit
Übersichtstabelle, einer Liste der Auffälligkeiten und Details je Host
(Dateisysteme, RAM, ausstehende Updates, fehlgeschlagene systemd-Units).
Das Verzeichnis `reports/` ist bewusst nicht eingecheckt.

## Codequalität

Jeder Push auf `main` und jeder Pull Request dagegen wird von GitHub Actions
geprüft (`.github/workflows/lint.yml`): `yamllint --strict`, `ansible-lint`
und ein `--syntax-check` über alle Playbooks.

Lokal derselbe Lauf:

```bash
pip install -r requirements-dev.txt
yamllint --strict .
ansible-lint --offline
```

Die Werkzeugversionen stehen in `requirements-dev.txt` auf Patch-Ebene
gepinnt. `ansible-lint` bringt mit neuen Minor-Versionen regelmäßig
zusätzliche Regeln mit — die sollen die CI nicht unvermittelt rot färben,
sondern beim bewussten Anheben des Pins auffallen.

`ansible-lint` ist auf das Profil `production` festgenagelt — das strengste
Profil, das das Repo aktuell erfüllt. Zwei Regeln sind in `.ansible-lint`
bewusst abgeschaltet, jeweils mit Begründung an Ort und Stelle:

- **`var-naming[no-role-prefix]`** — die Regel setzt voraus, dass Rollen
  unabhängige Einheiten sind. Hier teilen sie sich absichtlich Variablen
  (`maint_*`), und Namen wie `pve_snapshot_keep` sind die dokumentierte
  Bedienoberfläche.
- **`run-once`** — warnt vor `strategy: free`, die hier nirgends verwendet
  wird. `run_once` ist an jeder Stelle beabsichtigt.

## Aufbau

```
.github/workflows/          CI: yamllint, ansible-lint, Syntax-Check
ansible.cfg                 Grundeinstellungen
inventory/
  hosts.yml                 Hosts und Gruppen
  group_vars/               Einstellungen je Gruppe
playbooks/                  Die aufrufbaren Playbooks
  preflight.yml             Voraussetzungscheck vor dem ersten Lauf
roles/
  pve_facts                 Löst Gäste zu vmid/Node/Typ auf
  pve_snapshot              Snapshot anlegen und alte aufräumen
  pve_node_setup            Grundeinrichtung eines neuen Proxmox-Nodes
  pve_host_update           Updates für die PVE-Hosts selbst
  guest_update              Paket-Updates + Reboot-Erkennung
  guest_reboot              Neustart (LXC über den Host, VMs von innen)
  guest_cleanup             Paketreste, APT-Cache, Journal, Docker
  guest_health              Lesende Bestandsaufnahme + Report
  docker_install            Docker CE, rootful oder rootless
  docker_update             Docker selbst aktualisieren, mit Container-Kontrolle
  icinga_server             Icinga2-Stack als Docker-Verbund
  icinga_remote             Monitoring-Zugang und Plugins auf den Gästen
  icinga_config             Host-/Service-Definitionen aus dem Inventory
  ssh_keys                  Schlüssel ausrollen, optional sshd härten
```
