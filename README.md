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

## Aufbau

```
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
```
