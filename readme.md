# k8s-course

Source: https://www.linkedin.com/learning/kubernetes-grundkurs

#### Verbindung mit Pods
Verbindung zu Pods ist möglich, aber Änderungen betreffen stets eine Pod Instanz.
Somit sollten nur Lesezugriffe durchgeführt werden.
Mit `kubectl exec` kann Code im Container ausgeführt werden.
Beispiele:
```bash
#  Auflisten des working dirs
kubectl exec linkedin-timer-pod -- ls -la
# Shell im Pod öffnen
kubectl exec linkedin-timer-pod -it -- /bin/sh
```

#### Init-Container

https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

Spezielle Variante eines Container im Kubernetes-Umfeld, welcher genutzt wird um den eigentlichen Pod zu initialisieren.
Typische Anwendungsfälle sind: Datenbank-Updates, Set-up von Komponenten, Überprüfen von Stammdaten, Updates, etc.
Der eigentliche Container/Pod wird erst nach erfolgreicher Ausführung des Init-Containers gestartet.
Init-Container werden nur bei erneutem Deployment oder bei geändertem Init-Container-Image erneut ausgeführt.
Sind mehrere Init-Container definierte, werden diese in der Reihenfolge ihrere Definition ausgeführt.
Init-Container werden im Init-Container Bereich der YAML definiert.
Beispiel Konfiguration:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: linkedin-pod
spec:
    initContainers:
    - name: init-container
      image: busybox
      command: ['sh', '-c', 'echo Init Container! && sleep 10']
    containers:
    - name: linkedin-container
      image: busybox
      command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 1800']
```
### 2025-04-16

#### korrekte kubectl Berechtigung

Quelle:  https://0to1.nl/post/k3s-kubectl-permission/

`k3s` Konfiguration in die User Konfiguration kopieren und Pfad zu dieser in der `KUBECONFIG` Umgebungsvariable setzen.

```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && chown $USER ~/.kube/config && chmod 600 ~/.kube/config && export KUBECONFIG=~/.kube/config
```

Anschließend den `export` in der `.zshrc` sichern.
#### Pod

https://kubernetes.io/docs/concepts/workloads/pods/
https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/

In Kubernetes können Container nicht direkt bereitgestellt werden. Stattdessen erfolgt die Ausführung über sogenannte **Pods**, die die kleinste und zugleich atomare Deployment-Einheit innerhalb von Kubernetes darstellen. Ein Pod repräsentiert damit ein Container-Deployment und beinhaltet nicht nur einen oder mehrere Container, sondern auch sämtliche Meta-Informationen, die für deren Ausführung notwendig sind.

Die Beschreibung eines Pods erfolgt in der Regel über eine **Manifest-Datei im YAML-Format**, in der der gewünschte Zustand des Deployments definiert wird. Administratoren spezifizieren in dieser Datei, wie die Container im Pod aussehen sollen, welche Ressourcen sie benötigen und wie sie sich verhalten sollen – Kubernetes sorgt anschließend dafür, dass dieser gewünschte Zustand eingehalten wird.

Als **Orchestrator** übernimmt Kubernetes dabei vielfältige Aufgaben: Es entscheidet, auf welchem Knoten ein Pod ausgerollt wird, überwacht kontinuierlich die System- und Pod-Gesundheit, beendet nicht funktionierende Pods und startet sie bei Bedarf neu. Ziel ist es, den gewünschten Zustand der Umgebung konstant sicherzustellen – ganz ohne manuelles Eingreifen.

**Pods** selbst verfügen über Mechanismen zur Statusberichterstattung und können Metriken sowie Logs exponieren, um eine Überwachung und Analyse zu ermöglichen. Da alle internen Informationen beim Beenden eines Pods verloren gehen, müssen wichtige Daten stets extern gespeichert werden, zum Beispiel über Persistent Volumes.

Zusammenfassend lässt sich sagen: Kubernetes übernimmt das komplette Management von Pods. Administratoren müssen lediglich definieren, **was** erreicht werden soll – **wie** dieses Ziel erreicht wird, regelt Kubernetes automatisch.
##### Roll-out Optionen
###### Direktes Deployment
```bash
sudo k3s kubectl run linkedin-pod-cmd --image busybox -- /bin/sh -c "echo 'Hello Kubernetes' && sleep 1800"
```
###### Konfiguration per YAML Datei
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: linkedin-pod
spec:
    containers:
    - name: linkedin-container
      image: busybox
      command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 1800']

```
Anwenden der Konfiguration:
```bash
sudo k3s kubectl apply -f pod.yaml
```
Da die YAML-Datei versioniert werden kann, ist diese Deployment zu bevorzugen.
##### Verwaltung
`kubectl` erlaubt die Verwaltung von Pods, die Verwaltungs-Operationen sind `get`, `describe`, `edit` und `apply`.

```bash
# Mehr Informationen zu den Pods erhalten
sudo k3s kubectl get pod -o wide
# Kurzschreibweise von pod
sudo k3s kubectl get po
kgp
# zum beobachten --watch Option nutzen
kgp --watch
# Status eines Pods im yaml Format
sudo k3s kubectl get pod linkedin-pod -o yaml
# Besser lesbarer Status des Pods
sudo k3s kubectl describe pod linkedin-pod
# Editieren der Metadaten eines Pods
sudo k3s kubectl edit pod linkedin-pod
```
Mit der `edit` Operation kann lediglich die Container Version angepasst werden.
Das anpassen der yaml-Konfiguration und dem erneuten `apply` hat den selben Effekt.
##### Löschen
Nach dem Löschen werden sämtlich Ressourcen freigegeben, auch Dateien und Logs.
Das löschen ist eine synchrone Aktion und dauert daher einen Moment.
```bash
# Löschen eines Pods
sudo k3s kubectl delete pod linkedin-pod-cmd
```
##### Logs einsehen
Logs von Pods sind extern Abrufbar, dabei sind alle Standard- und Fehlerausgaben eines Pods verfügbar und das speichern der Logs in Dateien ist nicht notwendig.
Abruf über `kubectl logs <podname>`.

Mit `-f` können Logs beobachtet werden.
Beispiel yaml-Konfiguration:
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: linkedin-timer-pod
spec:
    containers:
    - name: linkedin-timer-container
      image: busybox:latest
      command: ['sh', '-c', 'for i in $(seq 1 1800); do echo $(date) && sleep 1; done']
```
Logt 30 Minuten jeden Sekunde das aktuelle Datum.
### 2025-04-15

##### Single Node Kubernetes
- Kubernetes auf nur einer Maschine
	- für gewöhnlich wird kubernetes auf mehreren Maschinen gleichzeitig betrieben
- ControlPlane-Node kann Workloads ausführen
	- kontrolliert den Status des Clusters, die kontrollierten Nodes (workloads) liegen außerhalb
- es gibt verschiedene Single Node Kubernetes Distributionen
	- k3s, Minikube, MicroK8s, selbst installiert

Habe jetzt [k3s](https://k3s.io/) installiert, in der Hoffnung das ich damit dem Kurs folgen kann.

- `kubectl` wird benötigt um mit einem Cluster zu interagieren.
- Versionen von `kubectl` und Cluster sollten übereinstimmen.
- Konfiguration für Cluster befindet sich in `~/.kube`