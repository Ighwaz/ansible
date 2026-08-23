# Ansible-Wartung für Proxmox VE

Playbooks zur Wartung von LXC-Containern und VMs auf Proxmox VE:
Snapshot anlegen, patchen, bei Bedarf neu starten, aufräumen und den Zustand
berichten.

Läuft mit reinem **ansible-core** — es müssen keine Collections installiert
werden. Die Proxmox-Ebene wird über `pvesh`, `pct` und `qm` direkt auf dem
PVE-Host angesprochen, es braucht also weder API-Token noch `proxmoxer` auf
dem Controller.

## Voraussetzungen

**Auf dem Controller** (dein Rechner): `ansible-core` ab 2.14.

**Auf den PVE-Hosts:** SSH-Zugang mit einem Benutzer, der per `sudo` root
werden darf. Nur darüber laufen `pvesh`, `pct` und `qm`.

**Auf den Gästen:** SSH-Zugang mit `sudo`-Rechten sowie `python3` (auf
Debian/Ubuntu ohnehin vorhanden). Ein sehr schlanker Container braucht ggf.
einmalig `apt install python3 sudo`.

## Einrichtung

1. `inventory/hosts.yml` an dein Setup anpassen — PVE-Hosts unter `proxmox`,
   Container unter `lxc`, VMs unter `vms`.
2. In `inventory/group_vars/all.yml` den `ansible_user` setzen.
3. Verbindung testen:

   ```bash
   ansible all -m ping
   ```

Eine `vmid` musst du **nicht** pflegen: Solange der Inventory-Name dem Namen
in Proxmox entspricht, löst Ansible vmid, Node und Typ automatisch über
`pvesh get /cluster/resources` auf. Weicht der Name ab, genügt ein
`pve_vmid: 123` beim betreffenden Host.

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
| `playbooks/pve-host-update.yml` | Die PVE-Hosts selbst aktualisieren | ja |
| `playbooks/docker-setup.yml` | Docker installieren (rootful oder rootless) | ja |
| `playbooks/docker-update.yml` | Docker selbst aktualisieren, mit Container-Kontrolle | ja |
| `playbooks/icinga-setup.yml` | Icinga2-Stack aufsetzen und Gäste vorbereiten | ja |
| `playbooks/icinga-config.yml` | Prüfungen aus dem Inventory neu erzeugen | ja |

### Vor dem ersten Lauf: Preflight

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
```
