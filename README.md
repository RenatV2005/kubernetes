# Kubernetes Homelab – IaC-Lernnotizen

> Zweck dieser Datei: Ich möchte später schnell nachvollziehen können, **was ich aufbaue, wieso  es so aufgebaut wird und welche Komponente welche Aufgabe haben**. Diese Notizen beschreiben den  bisherigen Stand und den geplanten Ablauf. Sie sind kein Beweis dafür, dass alle Schritte bereits erfolgreich getestet wurden, es ist ein kleines K8S Roadmap für mich.

## 1. RoadMap

Wir wollen einen Kubernetes-Cluster reproduzierbar als **Infrastructure as Code (IaC)** aufbauen. Anstatt jeden Server manuell einzurichten und Befehle spontan auszuführen, halten wir Konfigurationen und Automatisierung in Git fest.

Unser angestrebter Ablauf:

1. **Terraform** erstellt die VMs und die zugrunde liegende Infrastruktur.
2. **Ansible** bereitet Ubuntu vor und installiert die benötigte Software.
3. **kubeadm** initialisiert die Kubernetes-Control-Plane.
4. **Flannel** wird als CNI-Netzwerklösung im Cluster bereitgestellt.
5. Die Worker-Nodes treten dem Cluster bei.
6. Ansible prüft, ob die Komponenten tatsächlich funktionieren.

Wichtig: Das ist unser Zielbild. Nicht alle Schritte sind bereits vollständig automatisiert oder erfolgreich getestet.

---

## 2. Was macht welche Komponente?

| Komponente | Aufgabe |
|---|---|
| Terraform | Erstellt und verwaltet Infrastruktur, beispielsweise VMs. |
| Ansible | Konfiguriert Betriebssysteme, installiert Pakete und orchestriert Abläufe. |
| containerd | Container-Runtime, die Container startet und verwaltet. |
| kubeadm | Initialisiert die Control-Plane und unterstützt das Hinzufügen von Nodes. |
| kubelet | Node-Agent, der mit der Kubernetes-Control-Plane kommuniziert und Pods auf dem Node verwaltet. |
| kubectl | CLI-Client, mit dem wir mit der Kubernetes-API interagieren. |
| Kubernetes API Server | API-Einstiegspunkt für die Verwaltung von Kubernetes-Ressourcen. |
| Flannel | CNI-Netzwerklösung, die das Pod-Netzwerk zwischen Nodes ermöglicht. |
| Git | Versioniert unsere Templates, Manifeste, Variablen und Playbooks. |

### Ein wichtiger Unterschied

**YAML ist ein Dateiformat, kein Programm.** Was eine YAML-Datei bewirkt, hängt davon ab, welches Werkzeug sie liest und verarbeitet.

- `kubeadm-config.yaml`: wird von `kubeadm` als Konfiguration für den Cluster-Bootstrap gelesen.
- `kube-flannel.yml`: beschreibt Kubernetes-Ressourcen. Diese werden über die Kubernetes-API bereitgestellt, beispielsweise mit `kubectl apply` oder einem Ansible-Modul.

Die beiden Dateien sind also nicht austauschbar und werden nicht automatisch vom selben Programm verarbeitet.

---

## 3. Unsere Nodes

| Rolle | Hostname | IP-Adresse |
|---|---|---|
| Control Plane | `k8s-control-01` | `192.168.0.210` |
| Worker | `k8s-worker-01` | `192.168.0.211` |
| Worker | `k8s-worker-02` | `192.168.0.212` |

Diese IP-Adressen gehören zu unserer aktuellen Homelab-Umgebung. Für andere Umgebungen sollten wir umgebungsspezifische Werte in Variablen halten, statt sie an vielen Stellen fest einzutragen.

---

## 4. Repository-Struktur und bisherige Dateien

Die Struktur, die wir bisher gesehen bzw. angelegt haben:

```text
~/kubernetes/
├── README.md
├── cluster/
│   ├── kubeadm-config.yaml
│   └── networking/
│       └── flannel/
│           └── kube-flannel.yml
└── ansible/
    ├── ansible.cfg
    ├── inventory/
    │   └── kubernetes.ini
    ├── group_vars/
    │   └── kubernetes_control_plane.yml
    ├── site.yml
    └── playbooks/
        ├── kubernetes-bootstrap.yml
        ├── kubernetes-kernel.yml
        ├── kubernetes-network.yml
        ├── kubernetes-swapoff.yml
        ├── kubernetes-prefligt.yml
        ├── kubernetes-bootstrap-status.yml
        └── kubernetes-kubeadm-config-deploy.yml
```

**Hinweis zu Dateinamen und aktuellem Zustand:** Die Liste ist aus unserem bisherigen Gespräch zusammengesetzt. Wir haben nicht zuletzt erneut alle Dateien des Repositorys ausgelesen. Kontrolliere deshalb die tatsächlichen Dateinamen mit `find` oder `tree`, bevor du dich auf diese Liste verlässt. Wir haben auch eine Datei `cluster/kubeadm-config.yaml.j2` erstellt. Der neue Bootstrap-Entwurf referenziert dieses Template; die bestehende `group_vars`-Datei muss noch korrekt von Ansible gefunden werden.

### Wichtige Dateien

- `ansible/inventory/kubernetes.ini`: enthält die Hostgruppen, IP-Adressen und SSH-Verbindungsparameter.
- `ansible/ansible.cfg`: legt Standardwerte für Ansible fest, unter anderem das Inventory-Verzeichnis.
- `ansible/site.yml`: soll die grundlegende Vorbereitung der Nodes orchestrieren.
- `cluster/kubeadm-config.yaml`: bisherige statische kubeadm-Konfiguration.
- `cluster/kubeadm-config.yaml.j2`: Jinja2-Vorlage, die wir für Variablen verwenden wollen.
- `cluster/networking/flannel/kube-flannel.yml`: lokal gespeichertes Flannel-Manifest.
- `ansible/group_vars/kubernetes_control_plane.yml`: vorgesehene Gruppe von Variablen für die Control Plane. Ansible lädt `group_vars` automatisch abhängig von der Verzeichnisstruktur; wir müssen sicherstellen, dass diese Datei am richtigen Ort liegt.

---

## 5. Inventory: Woher weiß Ansible, welchen Server es ansprechen soll?

Beispiel aus unserem Inventory:

```ini
[kubernetes_control_plane]
k8s-control-01 ansible_host=192.168.0.210 ansible_user=renatk8s

[kubernetes_workers]
k8s-worker-01 ansible_host=192.168.0.211 ansible_user=renatk8s
k8s-worker-02 ansible_host=192.168.0.212 ansible_user=renatk8s

[kubernetes:children]
kubernetes_control_plane
kubernetes_workers
```

- `kubernetes_control_plane` ist eine Hostgruppe.
- `kubernetes_workers` ist eine weitere Hostgruppe.
- `kubernetes:children` enthält beide Gruppen.
- `ansible_host` gibt die IP-Adresse des Hosts an.
- `ansible_user` legt den SSH-Benutzer fest.

Ein Playbook mit `hosts: kubernetes` läuft auf allen drei Nodes. Ein Playbook mit `hosts: kubernetes_control_plane` läuft nur auf der Control Plane.

---

## 6. `playbook_dir`: Wie werden Dateipfade aufgelöst?

Ansible stellt `playbook_dir` automatisch bereit. Es enthält den Pfad zum Verzeichnis des ausgeführten Playbooks; wir müssen diese Variable nicht selbst definieren.

Wenn das Playbook unter

```text
~/kubernetes/ansible/playbooks/
```

liegt, kann dieser Quellpfad zur Vorlage zurück ins Repository führen:

```yaml
src: "{{ playbook_dir }}/../../cluster/kubeadm-config.yaml.j2"
```

Die zwei `..`-Segmente gehen von `ansible/playbooks/` über `ansible/` zurück nach `kubernetes/`.

Zum Überprüfen:

```bash
cd ~/kubernetes
realpath ansible/playbooks/../../cluster/kubeadm-config.yaml.j2
ls -l cluster/kubeadm-config.yaml.j2
```

Der Pfad muss auf die vorhandene Datei zeigen.

---

## 7. YAML, Jinja2 und Variablen

### YAML

YAML ist das Format, in dem wir strukturierte Daten und Konfigurationen notieren. Einrückungen sind wichtig.

### Jinja2-Template

Eine Datei mit der Endung `.j2` ist eine Vorlage, die Ansible rendern kann. Zum Beispiel:

```yaml
localAPIEndpoint:
  advertiseAddress: "{{ control_plane_ip }}"
```

`control_plane_ip` ist eine Variable. Beim Rendern ersetzt Ansible den Platzhalter durch ihren Wert.

### Variablen

Ein mögliches Beispiel für Werte, die wir zentral pflegen könnten:

```yaml
control_plane_ip: "192.168.0.210"
kubernetes_version: "v1.37.1"
pod_subnet: "10.244.0.0/16"
```

Das ist ein Architekturbeispiel und noch kein Nachweis dafür, dass alle diese Variablen bereits in der Vorlage verwendet werden. Wir müssen die Variablennamen zwischen Template und Variablendatei exakt abstimmen.

### `template` vs. `copy`

- `ansible.builtin.template`: rendert eine Jinja2-Vorlage und überträgt das Ergebnis.
- `ansible.builtin.copy`: überträgt eine Datei unverändert.

Für `kubeadm-config.yaml.j2` verwenden wir `template`, weil wir Variablen einsetzen wollen. Für das bereits fertige Flannel-Manifest verwenden wir normalerweise `copy`, wenn wir es erst auf den Control-Plane-Host übertragen wollen.

---

## 8. `kubeadm-config.yaml`: Wie wird der Cluster initialisiert?

Die kubeadm-Konfiguration beschreibt Parameter für den Bootstrap. In unserer bisherigen Konfiguration sind unter anderem vorgesehen:

- API-Server-Adresse: `192.168.0.210`
- API-Port: `6443`
- Kubernetes-Version: `v1.37.1`
- Pod-CIDR: `10.244.0.0/16`
- Container-Runtime-Socket: `unix:///run/containerd/containerd.sock`

Ein Bootstrap-Befehl sieht prinzipiell so aus:

```bash
kubeadm init --config /etc/kubernetes/kubeadm-config.yaml
```

Dieser Befehl soll auf einer frisch vorbereiteten Control Plane ausgeführt werden. Er darf nicht blind erneut auf einer bereits initialisierten Control Plane laufen.

**Achtung:** Wir müssen die konkrete Konfiguration vor dem Einsatz validieren und mit der tatsächlichen VM-Konfiguration abgleichen. Das Vorhandensein einer YAML-Datei bedeutet nicht automatisch, dass die Werte richtig oder angewendet sind.

---

## 9. Was ist Flannel, und was macht `kube-flannel.yml`?

Flannel ist eine CNI-Netzwerklösung für Kubernetes. CNI steht für **Container Network Interface**.

In unserem Flannel-Manifest sehen wir unter anderem:

- `Namespace`: organisiert Flannels Ressourcen im Namespace `kube-flannel`.
- `ServiceAccount`, `ClusterRole` und `ClusterRoleBinding`: geben Flannel die benötigten API-Rechte.
- `ConfigMap`: enthält Netzwerkeinstellungen, darunter `10.244.0.0/16` und das VXLAN-Backend.
- `DaemonSet`: sorgt dafür, dass Flannel auf geeigneten Nodes läuft.
- `initContainers`: kopieren benötigte CNI-Dateien auf den jeweiligen Host.
- `image`: benennt die Container-Images, die containerd herunterladen muss.

Flannel ist also **nicht nur ein Programm, das ausschließlich auf der Control Plane läuft**. Der Flannel-Daemon läuft als Pod auf geeigneten Nodes. Zusätzlich werden CNI-Plugins und Konfigurationsdateien auf den Hosts bereitgestellt.

### Wie wird das Manifest angewendet?

Ein manueller Vergleichsbefehl wäre:

```bash
kubectl apply -f cluster/networking/flannel/kube-flannel.yml
```

Unser Ziel ist, diesen Schritt durch Ansible zu automatisieren. Ansible kann die Datei auf die Control Plane kopieren und danach `kubectl apply` ausführen oder ein Kubernetes-Modul verwenden.

Das ist ein separater Schritt nach der Initialisierung und Erreichbarkeit der Kubernetes-API. `kubeadm init` installiert dieses Manifest nicht automatisch.

### Warum speichern wir das Manifest lokal?

Wir wollen keine unversionierte `latest`-URL bei jedem Aufbau verwenden. Durch das lokale Manifest können wir Änderungen mit Git nachvollziehen und dieselbe deklarierte Konfiguration wiederverwenden. Die Version und Integrität der Datei sollten wir ebenfalls bewusst verwalten.

---

## 10. Der geplante Ablauf auf einer frischen Umgebung

### Phase A – Infrastruktur

1. Terraform erstellt die Ubuntu-VMs.
2. Netzwerk und SSH-Zugang werden überprüft.

### Phase B – Node-Vorbereitung

1. `ansible/site.yml` bereitet die Nodes vor.
2. Swap wird deaktiviert.
3. Kernel-Module und Netzwerkparameter werden gesetzt.
4. containerd und Kubernetes-Pakete werden installiert.
5. Dienste und Versionen werden geprüft.

### Phase C – Control-Plane-Bootstrap

1. Ansible rendert das kubeadm-Template.
2. Ansible legt die resultierende Datei auf der Control Plane ab.
3. Ansible prüft, ob die Control Plane bereits initialisiert wurde.
4. `kubeadm init --config ...` läuft nur, wenn der Bootstrap noch nicht erfolgt ist.
5. Ansible wartet auf die Erreichbarkeit der Kubernetes-API.

### Phase D – CNI

1. Ansible stellt das versionierte Flannel-Manifest bereit.
2. Das Manifest wird an die Kubernetes-API übermittelt.
3. Kubernetes erstellt die Flannel-Ressourcen.
4. Ansible prüft den Zustand der Flannel-Pods und Nodes.

### Phase E – Worker

1. Die Worker werden vorbereitet.
2. Ein Join-Ablauf verwendet kontrolliert einen gültigen `kubeadm join`-Befehl bzw. ein geeignetes automatisiertes Verfahren.
3. Ansible prüft, ob alle Nodes `Ready` werden.

### Phase F – Abnahmetests

- Alle erwarteten Nodes sind `Ready`.
- Flannel-Pods laufen auf den geeigneten Nodes.
- Pod-Netzwerk und DNS funktionieren.
- Ein einfacher Test-Pod kann gestartet werden und kommunizieren.

**Dieser vollständige Ablauf ist unser Ziel. Er ist noch nicht komplett implementiert und getestet.**

---

## 11. Wie wir Playbooks testen

Syntax prüfen:

```bash
ansible-playbook --syntax-check ansible/playbooks/kubernetes-cluster-bootstrap.yml
```

Playbook simulieren:

```bash
ansible-playbook --check --diff ansible/playbooks/kubernetes-cluster-bootstrap.yml
```

Der Check-Modus ist nur eine Simulation. Er garantiert nicht, dass alle Befehle und externen Änderungen wie bei einem echten Lauf funktionieren.

Ansible-Lint verwenden:

```bash
ansible-lint ansible/playbooks/kubernetes-cluster-bootstrap.yml
```

Vor dem Ausführen auf einer neuen VM sollten wir außerdem prüfen:

- Stimmt das Inventory?
- Funktioniert SSH mit dem vorgesehenen Benutzer?
- Ist `sudo` verfügbar?
- Existieren Template und Manifest am erwarteten Pfad?
- Stimmen die Variablennamen?
- Ist die VM wirklich frisch oder existiert bereits ein Kubernetes-Zustand?

---

## 12. Was wir bisher gelernt und gemacht haben

- Ein Ansible-Inventory mit Control Plane und Worker-Nodes ist vorhanden.
- Mehrere Playbooks für Kernel-Vorbereitung, Swap und Kubernetes-Paketinstallation existieren.
- Wir haben eine kubeadm-Konfigurationsdatei und eine Jinja2-Vorlage vorbereitet.
- Wir haben `group_vars` angelegt; der tatsächliche Speicherort muss geprüft werden.
- Wir haben das offizielle Flannel-Manifest lokal in `cluster/networking/flannel/kube-flannel.yml` gespeichert.
- `ansible-lint` hat für das damalige Konfigurations-Deploy-Playbook null Fehler und null Warnungen gemeldet.
- Wir haben die Trennung zwischen VM-Provisionierung, Node-Vorbereitung, Cluster-Bootstrap und CNI-Deployment herausgearbeitet.

### Bekannte offene Punkte

- Sicherstellen, dass die Jinja2-Vorlage und Variablendatei korrekt zusammenpassen.
- Den tatsächlichen Zustand der neu erstellten Control Plane prüfen.
- Den Bootstrap-Ablauf auf der frischen VM sicher testen.
- Flannel-Deployment durch Ansible sauber umsetzen.
- Worker-Join automatisieren.
- End-to-End-Tests ergänzen.
- Die Terraform-Neuerstellung erst dann als zerstörenden Test durchführen, wenn der Wiederaufbau auf einer kontrollierten Testumgebung funktioniert.

---

## 13. Die wichtigsten Merksätze

1. **Terraform erstellt die Infrastruktur.**
2. **Ansible konfiguriert die Nodes und orchestriert Abläufe.**
3. **kubeadm initialisiert den Kubernetes-Cluster.**
4. **`kubeadm-config.yaml` ist die Eingabekonfiguration für kubeadm.**
5. **`kube-flannel.yml` beschreibt Kubernetes-Ressourcen, die über die Kubernetes-API bereitgestellt werden.**
6. **Flannel ist eine CNI-Netzwerklösung, deren Daemon als Pod auf geeigneten Nodes läuft.**
7. **YAML führt selbst nichts aus. Entscheidend ist, welches Werkzeug die Datei verarbeitet.**
8. **Eine versionierte Konfiguration ist erst dann wirklich reproduzierbar, wenn der gesamte Ablauf getestet und dokumentiert wurde.**
9. **Vor einem erneuten `kubeadm init` immer prüfen, ob der Cluster bereits initialisiert wurde.**
10. **Vor einem destruktiven Terraform-Test sicherstellen, dass der Neuaufbau funktioniert und keine benötigten Daten verloren gehen.**

---

## 14. Schnellstart: Wo mache ich weiter?

Wenn ich das Projekt wieder aufnehme, gehe ich in dieser Reihenfolge vor:

1. Repository-Struktur und Dateinamen prüfen.
2. Inventory, `group_vars` und Template-Pfade kontrollieren.
3. Zustand der frisch erstellten Control Plane mit einem read-only Preflight prüfen.
4. kubeadm-Konfiguration validieren.
5. Bootstrap-Playbook prüfen und erst dann auf der frischen VM ausführen.
6. Flannel aus dem versionierten Manifest bereitstellen.
7. Worker-Join automatisieren.
8. Cluster- und Netzwerktests durchführen.
9. Erst danach den kompletten Terraform-Rebuild testen.

**Grundprinzip:** Erst verstehen und prüfen, dann automatisieren, dann reproduzierbar testen.
