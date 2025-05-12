# k8s-course

Source: https://www.linkedin.com/learning/kubernetes-grundkurs

---

### 2025-05-09

#### StorageClass

Parameter der `StorageClass`:
- `Provisioner` legt fest welcher `Provisioner` verwendet werden soll.
- `ReclaimPolicy` legt fest, wie mit nicht mehr gebundene, Speicher umgegangen werden soll.
- `VolumeBindingMode` legt fest wann `Volumes` gebunden werden.
- `AllowVolumeExpansion` legt fest, ob der Speicher eines `Volumes` dynamisch vergrößert werden darf.

Definition:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: linkedin-storageclass
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: false
reclaimPolicy: Delete
```

Können wie andere k8s Objekte mit `k apply` ausgerollt werden.
Typische Befehle:

```bash
# Ausrollen
k apply -f <datei.yaml>
# Auflisten
k get sc
```

##### Definition PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: linkedin-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: linkedin-storageclass
  resources:
    requests:
      storage: 1Gi
```

Wie jedes andere k8s Objekt mit `k apply` anzuwenden.

Das `Deployment` für den `PersistentVolumeClaim`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkedin-persistent-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: linkedin-persistenz
  template:
    metadata:
      labels:
        app: linkedin-persistenz
    spec:
      containers:
      - name: app
        image: busybox:latest
        command: ["/bin/sh", "-c", "eval echo 'Hello from $(hostname)!' > /data/$(hostname).txt && sleep infinity"]
        volumeMounts:
        - name: linkedin-data-volume
          mountPath: /data
      volumes:
      - name: linkedin-data-volume
        persistentVolumeClaim:
          claimName: linkedin-data-pvc

```

Das `PersistentVolumeClaim` wird von  k8s erst genutzt, wenn dieses von einem `Deployment` verwendet wird.
`k get pv` listet die vorhandenen `PersistentVolumeClaim` auf, der Status `BOUND` gibt an  das es verwendet wird.

`Provisioner`, `StorageClass` und `PersistentVolumeClaim` sind Voraussetzungen für persistente Datenspeicherung mit Kubernetes.
`PVC` wird in Pod-Definitionen eingebunden und dann per `Volume-Mount` in Container verfügbar gemacht.

`Local-Path-Provisioner` speichert Daten in `/opt/local-path-provisioner` (oder wenn k3s genutzt wird in `/var/lib/rancher/k3s/storage`) ab.
### 2025-05-08
💭 es macht Sinn wenn ich mir Literatur Notizen aus meinen Unterlagen erzeuge, bevor ich die ersten Projekte starte. (Pod, Services usw.)
### 2025-05-07 & 2025-05-06

#### Secrets
Funktionieren genau wie `ConfigMaps`.
Informationen können wiederverwendbar verschlüsselt verwaltet werden.

- `k get secrets` gibt alle `Secrets` aus.
- `k describe secrets` beschreibt `Secrets` detailliert, die Umgebungsvariablen werden nicht angezeigt, nur deren länge.

#### Definition eines Secrets

```yaml
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: linkedin-secret
data:
  message: "RWluIHZlcnNjaGlzc2VuZXMgR2VoZWltbmlzCg=="
  autor: "RGFuaWVsIE5hZ2VsCg=="
# ---
# Pod welcher Secret als Umgebungsvariable referenziert.
# Schlüsselwort envFrom.secretRef für das referenzieren
# der Secret als Umgebungsvariable.
apiVersion: v1
kind: Pod
metadata:
  name: linkedin-secret-env
spec:
  containers:
    - name: busybox
      image: busybox:latest
      command: ["/bin/sh", "-c", "env && sleep 3600"]
      envFrom:
        - secretRef:
            name: linkedin-secret
# ---
# Pod welcher Secret als Datei referenziert.
# Es muss ein Volume angegeben werden, welches das Secret referenziert.
# Mit volumeMounts wird der Pfad angegeben,
# in dem die Secrets im Container gesichert werden soll.
apiVersion: v1
kind: Pod
metadata:
  name: linkedin-secrets-file
spec:
  containers:
    - name: busybox
      image: busybox:latest
      command: ["/bin/sh", "-c", "ls /etc/secrets && sleep 3600"]
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
  volumes:
    - name: secret-volume
      secret:
        secretName: linkedin-secret
```

#### Volumes
https://kubernetes.io/docs/concepts/storage/volumes/
##### Datenaustausch von Pods

Grundsätzlich gilt, Daten sind stest an die Instanz gebunden und gehen verloren sobald der Pod gelöscht wird.
Werden auf Pod Level definiert.
`Volumes` erlauben den dateisystembasierenden Datenaustausch zwischen Pods.
Dabei gibt es verschiedene Arten von `Volumes`, z.B.:
- [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) ist ein anfänglich leeres Verzeichnis, das mit der Zeit neue Daten aufnehmen kann.
- [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) verweist auf ein Verzeichnis auf der Maschine (`local` wird aus Sicherheitsgründen empfohlen)
- `configMaps` und `Secrets`
`Volumes` teilen sich den Lebenszyklus mit Pods, sobald der letze Pod, mit Zugriff auf das `Volume`, abgeschaltet ist, wird auch das `Volume` freigegeben.
Volumes sind nur für den kurzen temporären Datenaustausch gedacht.

Definition:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkedin-volume-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linkedin-volume
  template:
    metadata:
      labels:
        app: linkedin-volume
    spec:
      containers:
      - name: webserver
        image: nginx:alpine
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      - name: data-writer
        image: busybox:latest
        command: ["/bin/sh", "-c", "while true; do echo \"It's $(date)\" > /data/index.html; sleep 1; done"]
        volumeMounts:
        - name: shared-data
          mountPath: /data
      volumes:
      - name: shared-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: linkedin-volume-demo
spec:
  selector:
    app: linkedin-volume
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

#### Persistent Volumes
https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Daten in Pods sind flüchtig. Eine Persistente Speicherung hält Daten über Pod-Lebenszyklen hinweg verfügbar.
Persistente Speicherorte (lokal, Netzwerk, Cloud) werden mit Hilfe von Volumes abstrahiert.
Volume Mounts machen Volumes in Container verfügbar.

🧠 Das Arbeiten mit Kubernetes im Heimnetzwerk hat folgenden Unterschied:
- man Arbeit mit flüchtigen Containern und kann diese einfach verwalten.
- durch die Abstrkation ist das Setup schwerer zu verstehen.
Was ich mir wünsche ist eine einfache, wiederherstellbare, versionierbare Konfiguration.
Allerdings laufen meine Backup System bereits, wofür brauche ich also ein Homelab?
- virtualisieren
- datenaustausch
- backups

`Persistent Volumes` Definition und Lebenszyklus sind unabhängig vom Pod.
Dadurch erlauben sie eine dauerhafte Datenspeicherung.

Es gibt weitere Arten von Komponente, welche k8s zur Datenspeicherung bereit stellt:
- `Provisioner`: stellt Speicher bereit..
- `StorageClass`: vordefinierte Konfiguration https://kubernetes.io/docs/concepts/storage/storage-classes/
- `PersistenVolume`: repräsentiert einen Speicherort.
- `PersistentVolumeClaim`: Anfrage, Speicher bereit zu stellen.

##### local-path-provisioner

Bereitstellen:
`kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml`

Der `local-path-provisioner` ist die grunlegende Komponente um mit persistentem Speicher in Kubernetes arbeiten zu können.
### 2025-05-05 & 2025-05-06

#### Secrets
Funktionieren genau wie `ConfigMaps`.
Informationen können wiederverwendbar verschlüsselt verwaltet werden.

- `k get secrets` gibt alle `Secrets` aus.
- `k describe secrets` beschreibt `Secrets` detailliert, die Umgebungsvariablen werden nicht angezeigt, nur deren länge.

#### Definition eines Secrets

```yaml
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: linkedin-secret
data:
  message: "RWluIHZlcnNjaGlzc2VuZXMgR2VoZWltbmlzCg=="
  autor: "RGFuaWVsIE5hZ2VsCg=="
# ---
# Pod welcher Secret als Umgebungsvariable referenziert.
# Schlüsselwort envFrom.secretRef für das referenzieren
# der Secret als Umgebungsvariable.
apiVersion: v1
kind: Pod
metadata:
  name: linkedin-secret-env
spec:
  containers:
    - name: busybox
      image: busybox:latest
      command: ["/bin/sh", "-c", "env && sleep 3600"]
      envFrom:
        - secretRef:
            name: linkedin-secret
# ---
# Pod welcher Secret als Datei referenziert.
# Es muss ein Volume angegeben werden, welches das Secret referenziert.
# Mit volumeMounts wird der Pfad angegeben,
# in dem die Secrets im Container gesichert werden soll.
apiVersion: v1
kind: Pod
metadata:
  name: linkedin-secrets-file
spec:
  containers:
    - name: busybox
      image: busybox:latest
      command: ["/bin/sh", "-c", "ls /etc/secrets && sleep 3600"]
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
  volumes:
    - name: secret-volume
      secret:
        secretName: linkedin-secret
```

#### ConfigMaps
https://kubernetes.io/docs/concepts/configuration/configmap/
`ConfigMaps` sind über Pods und `Deployments` hinweg wiederverwendbare _Umgebungsvariablen_ bzw. externalisiert und wiederverwendbar Konfigurationen.
Diese werden in Pod-Definitionen abgefragt und können mehrfach verwendet werden.
Bei `ConfigMaps` handelt es sich um `Kubernetes` Objekte welche mit `kubetcl apply` bereit gestellt werden können.

`ConfigMaps` können als Umgebungsvariable oder als Datei eingebunden werden.
Alle Einträge stehen dann als einzelne Dateien zur Verfügung (Dateiname = Name des `ConfigMap`-Parameters).
Änderungen erfordern einen Neustart der nutzenden Pods!
Die Art der Einbindung wird nicht über die `ConfigMap` sondern den verwendenden Pod gesteuert.
Die Einbindung erfolgt Statisch, d.h. beim Start des Dateisystems.

##### Definition einer ConfigMap

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: linkedin-configmap
data:
  message: "Hallo von Kubernetes!"
  autor: "Daniel Nagel"
  KURS: "Kubernetes Grundkurs"
  KURS_URL: "https://www.linkedin.com/learning/kubernetes-grundkurs"
# ---
# Pod welcher ConfigMap als Umgebungsvariable referenziert.
# Schlüsselwort envFrom.configMapRef für das referenzieren
# der ConfigMap als Umgebungsvariable.
apiVersion: v1
kind: Pod
metadata:
  name: linkedin-configmap-env
spec:
  containers:
    - name: busybox
      image: busybox:latest
      command: ["/bin/sh", "-c", "env && sleep 3600"]
      envFrom:
        - configMapRef:
            name: linkedin-configmap

# ---
# Pod welcher ConfigMap als Datei referenziert.
# Es muss ein Volume angegeben werden, welches die ConfigMap referenziert.
# Mit volumeMounts wird der Pfad angegeben,
# an dem die ConfigMap im Container gesichert werden soll.
apiVersion: v1
kind: Pod
metadata:
  name: linkedin-configmap-file
spec:
  containers:
    - name: busybox
      image: busybox:latest
      command: ["/bin/sh", "-c", "ls /etc/config && sleep 3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: linkedin-configmap
```

- Mit `envFrom.configMapRef.name` kann ein Pod eine `ConfigMap` referenzieren.
- Mit `k get configmaps` können alle `ConfigMaps` angezeigt werden.
- Mit `k describe configmaps <configmap>` können die Details der `ConfigMap` eingesehen werden.
- Mit `k exec -it linkedin-confimap-file -- cat /etc/config/KURS` einen Befehl in einem Pod ausführen, in diesem Beispiel wird der Inhalt einer Datei ausgegeben.
### 2025-05-02

#### Unveränderliche Applikationen

Container-Images werden nur einmal gebaut, daher werden alle Konfigurationen extern gehalten (das gilt vor allem für Kennwörter und sensible Daten), das Prinzip heißt **externalisierte Konfiguration**.
Die Informationen werden erst zur Laufzeit zur Verfügung gestellt.

##### Arten von externalisierte Konfiguration

- `Umgebungsvariablen`
- `ConfigMaps`
	- über Pods und Deployments hinweg wiederverwendbare _Umgebungsvariablen_.
	- https://kubernetes.io/docs/concepts/configuration/configmap/
	- Es sollten keine sensiblen Daten gespeichert werden.
- `Secrets`
	- vertraulich Informationen (Kennwörter, Tokens, etc.) für Pods und Deployments sichern.
	- https://kubernetes.io/docs/concepts/configuration/secret/
- `Volumes`
	- Daten auf Festplatten oder in virtuellen Dateisystemen sichern.
	- https://kubernetes.io/docs/concepts/storage/volumes/
		- Eine Konfigurationsdatei, basierend auf ConfigMaps oder Secrets, im Datei System sichern.
		- Ein Dateisystem zwischen zwei Containern im selben Pod einrichten.
		- Ein Dateisystem zwischen zwei verschiedenen Pods, sogar auf unterschiedlichen Nodes, einrichten.
		- Daten über die Laufzeit eines Pods hinaus sichern.

##### Umgebungsvariablen verwenden

Beispiel YAML-Konfiguration:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: linkedin-env-demo
spec:
  containers:
    - name: busybox
      image: busybox:latest
      command: ["/bin/sh", "-c", "env && sleep 3600"]
      env:
        - name: NACHRICHT
          value: "Hallo von Kubernetes!"
```

### 2025-04-29 & 2025-04-30 & 2025-05-02

#### Ingress
https://kubernetes.io/docs/concepts/services-networking/ingress/
Ein `Ingress` ist eine spezielle Variante eines `Services`.
Zugriff nur via HTTP/HTTPS. Durch einen regelbasierten Ansatz werden weitere `Services` an den `Ingress` gebunden.
Es wird eine Load-Balancing Funktionalität angebunden und Ingress kann SSL Verbindungen Terminieren, wodurch diese unverschlüsselt innerhalb des Services weiterlaufen, was performanter ist.

![[regelbasierte-ingress-weiterleitung.png]]
Ein `Ingress` kann Anfragen zu einem bestimmten Pfad an einen `Service` weiterleiten, dieser spricht wiederum seinen entsprechenden `Pod` an.
Regeln können Pfade oder Domains oder eine Kombination aus beidem sein.
Die jeweilige Ingress Komponente muss installiert sein.
Kubernetes Ingress Controller sind austauschbar.

##### Ingress-Definition
In YAML spezifziert

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: linkedin-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service-a
            port:
              number: 80
```

##### nginx-Ingress ausrollen
https://github.com/kubernetes/ingress-nginx?tab=readme-ov-file
Am einfachsten über ein `HELM`-Chart.

```bash
# nginx repo installieren
helm repo add nginx https://kubernetes.github.io/ingress-nginx
# alle repos updaten
helm repo update
# suche nach nginx in allen repositories
helm search repo nginx
# installieren/upgrade des nginx/ingress-nginx Packets,
# im namespace ingress-nginx, welcher automatisch angelegt wird
helm upgrade --install ingress-nginx nginx/ingress-nginx --namespace ingress-nginx --create-namespace
# auflisten aller namespaces
k get ns
# auflisten aller pods im ingress-nginx namensraum
k get po -n ingress-nginx
```

##### traefik ingress ausrollen
https://doc.traefik.io/traefik/getting-started/install-traefik/#use-the-helm-chart
`Traefik` ist auch ein beliebter Ingress Controller und zeichnet sich durch eine besonders einfache Konfiguration und gut Performance aus. `Traeffik` wir wie der `nginx`-Ingress am einfachsten per `HELM`-Chart ausgerollt.

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm search repo traefik
helm upgrade --install traefik traefik/traefik --namespace ingress-traefik --create-namepsace
```

#### Ingress definieren
Es muss zuvor ein `Deployment` bereitgestellt und ein `Service` ausgerollt werden. Dann kann ein `Ingress` mit mindestens einer Regel definiert werden.

Mein Ingress hängt in einem Pending state fest, wie es aussieht muss ich Metallb auf einem lokalen (nicht Cloud) Umgebung installieren:
https://medium.com/@galvao.gabrielg/metallb-the-solution-for-external-ips-in-kubernetes-with-loadbalancer-learn-more-23fc2aff4e2b

Wenn ich es lokal Testen möchte geht es doch ich musste einfach über `localhost:<ingress-port>/test` zugreifen.

Immerhin habe ich jetzt eine offizielle Dokumentation zu k3s mit tailscale gefunden: https://docs.k3s.io/networking/distributed-multicloud
### 2025-04-28 & 2025-04-29

#### Services
https://kubernetes.io/docs/concepts/services-networking/service/
`Pods/Deployments` sind volatil und kurzlebig sind, können skaliert und terminiert werden, werden im Fehlerfall neu gestartet werden, wobei eine neue IP vergeben wird und sind somit flüchtige Komponenten.
Ein `Service` repräsentiert eine Menge an Pods, hat eine eindeutige und stabile IP, bietet Load-Balancer funktionalität und ist somit eine stabile Komponente.

`Pods` exponieren Labels und `Services` definieren für welche Services sie zuständig sind.
Beziehung zwischen `Pod` und `Service` wird durch Labels definiert.
![[fuktionsweise-von-k8s-services.png]]
Ein Service wird wie folgt definiert:

```yaml
# Service
apiVersion: v1
kind: Service
metadata:
  name: service-a
spec:
  type: ClusterIP
  selector:
    # label für pod-service kommunikation
    tier: service-a
  ports:
  - protocol: TCP
    port: 8000
    # pod port
    targetPort: 80

# Zum Service zugehöriger Pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    # label für pod-service kommunikation
    tier: service-a
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
    # pod port
    - containerPort: 80
      name: http-web-svc
```

##### Typen von Services
https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
- `ClusterIP` ein Service mit einer nur im Cluster gültigen IP-Adresse (am häufigsten verwendeter Service-Typ)
- `NodePort` Services sind über alle Nodes im Cluster verfügbar.
- `LoadBalancer` Services werden über einen externen Load Balancer bereit gestellt und können somit über eine virtuelle oder externe IP-Adresse angesprochen werden.
- `ExternalName` Services machen externe Komponenten im Cluster verfügbar.

##### Prinzip
Ein `Pod` definiert einen Labels. Ein `Service` nutzt Labels, um zu repräsentierende `Pods`zu identifizieren. Eingehende Anfragen werden dann per Load-Balancer auf die repräsentierten `Pods` verteilt.

##### Ausrollen eines Services
Ein `Deployment` **muss** vorher oder danach ausgerollt werden.
Die Reihenfolge, ob `Pods` oder `Service` zuerst ausgerollt werden spielt keine Rolle.
```bash
# Deployment im Cluster ausrollen
k apply -f configurations/deployment.yaml
# Service im Cluster ausrollen
k apply -f configurations/service.yaml
# Auflisten der Services
k get svc
# Beziehung zwischen Service und Pod betrachten
k describe svc nginx-service
```

### 2025-04-27

#### HELM
https://helm.sh/
`HELM` ist ein **package Manager**, dieser vereinfacht Installation, Upgrade und Verwaltung von Anwendungen. Anbieter können Repositories definieren, um dort ihre `Charts` abzulegen.
`Charts` sind YAML-basierte Beschreibungen von Kubernetes-Ressourcen.
Mit `Charts` lassen sich komplexe Applikationen bereitstellen, eine große Menge an vorgefertigten `Charts` ist verfügbar.
`HELM` erleichtert die Bereitstellung und Veraltung von Applikationen, organisiert `Charts` in Repositories, Werte von `Charts` können beim Rollout oder bei Updates angepasst werden und es ist ein sehr gerne genutztes Tool für k8s.

Beim `HELM` muss man zwischen drei Konzepten Unterscheiden:
- Ein `Chart` ist ein Helm Package. Es beinhaltet alle nötigen Definitionen um eine Applikation, Werkzeug oder Dienst im Kubernetes Cluster auszuführen.
- Ein `Repository` sammelt und verteilt `Charts` an einem Ort. So wie bei Pacakge Mangern, z.B. `apt` oder `npm`.
- Ein `Release` ist eine Instanz eines laufenden `Charts` innerhalb eines Kubernetes-Clusters. Ein `Chart` kann mehrfach im selben Cluster installiert werden, was wiederum neue `Releases`erzeugt.

Kurz gesagt: `Helm` installiert `Charts` in ein Kubernets-Cluster, welches neue `Releases` für jede Installation erzeugt. Neue Charts können in einem installierten Helm Chart `Repository` gesucht werden.

##### HELM installieren
https://helm.sh/docs/intro/quickstart/

##### Wichtige HELM-Befehle
- `helm repo add` fügt ein Repository hinzu.
- `helm repo update` lädt die Meta-Informationen von Repositories.
- `helm search repo` durchsucht Repositories.
- `helm install` installiert eine Applikation.
- `helm update` aktualisiert eine Applikation.
- `helm delete` löscht eine Applikation.
- `helm list` Listet die installierten Applikationen auf.
Parameter können inline mit `--set` überschrieben werden.

##### Applikation mit HELM bereitstellen
Alle Wordpress Parameter gibt es hier: https://github.com/bitnami/charts/blob/main/bitnami/wordpress/README.md#parameters
Kann auch mit `helm show values bitnami/wordpress` aufgelistet werden.

```bash
# repository installieren
# dieses Repository wird auch vom HELM Quickstart empfohlen
helm repo add bitnami https://charts.bitnami.com/bitnami
# repository aktualisieren
helm repo update
# Alle Pakete im Repository auflisten
helm search repo
# Nach einem Paket im Repository suchen
helm search repo wordpress
# Paket installieren, für Parameter des wordpress Pakets, siehe obigen Link.
helm install wordpress bitnami/wordpress --set mariadb.auth.rootPassword=sicher123 --set ingress.enabled=false --set service.type=NodePort --set readinessProbe.enabled=false --set livenessProbe.enabled=false
# auflisten der k8s Services, so können wir sehen,
# unter welchem Port der Webserver Wordpress ausliefert.
k get svc
# Listet die installierten Applikationen auf
helm list
# Zeigt die eingestellten, vom Standard abweichenden, Parameter
helm get values wordpress
# Deinstalliert das Release, alle Kubernetes-Ressourcen werden freigegeben
helm uninstall wordpress
```

### 2025-04-20 & 2025-04-21 & 2025-04-26

#### Deployments
https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
##### Nachteile von Replicasets

Replicatsets bieten grundlegende Funktionen rund um Skalierung und unterbrechungsfreie Updates, allerdings können Replicastes nicht erkenne, wie der Status eines Pods ist, ob ein Pod einsatzbereit ist und können Pods nicht auf ältere Stände zurückrollen.

##### Vorteile von Deployments

Deployments bauen auf Replicasets auf und bieten erweiterte Funktionalitäten rund um Gesundheitschecks, Update-Strategien und Rollbacks an.
Deployments werden von k8s verwaltet und in der Regel im YAML-Format definiert.

Definition eines Deployments:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```
##### Deployment ausrollen
Per CLI kann, unter der notwendigen Angabe des Deployment Namens und des Container-Images, mit dem Befehl `kubectl create deployment` ein Deployment ausgerollt werden.
Diese Art des Deployment ist aufgrund der nicht existenten Versionierung nicht empfehlenswert.

Die Eigenschaften `livenessProbe`und `readinessProbe` lassen sich nur in Deployments definieren.
`livenessProbe` legen fest, ob ein Pod noch erreichbar ist/ funktioniert.
`readinessProbe` stellt sicher ob ein Pod initial verfügbar ist.

Eine YAML-Defintion eines Deployments kann wie gehabt mit `kubectl apply -f <config.yaml>` gestartet werden.

##### Deployment skalieren
###### Manuell
Ein Deployment kann deklarativ durch die Modifikation der YAML-Datei oder direkt über die Kommandozeile skaliert werden.
Dabei wir die Anzahl der Pods unmittelbar geändert.
Dazu wird in der YAML-Datei die `replicas` Eigenschaft angepasst oder temporär per CLI mit dem Befehl `kubectl scale deployment`.
###### Automatisch
https://kubernetes.io/docs/concepts/workloads/autoscaling/
https://kubernetes.io/docs/tutorials/kubernetes-basics/scale/scale-intro/
Ein Deployment kann automatisch basierend auf Metriken, z.B. CPU-Auslastung, Speicherverbrauche, benutzerdefinierte Metriken, die Anzahl der Pods anpassen. Die Mindest- und Maximalanzahl der Pods und die Schwellwerte für die Skalierung sind konfigurierbar.
Das automatische Skalieren von Deployments erfolgt per `HorizontalPodAutoscaler` (beim horizontalen skalieren, siehe [auoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/)). Das ist entweder deklarativ per YAML-Datei ausrollbar oder einfacher über die Kommandozeile möglich `kubectl autoscale deployment nginx --max=5 --min=1 --cpu-percent=80`.

Beispiel YAML-Konfiguration:
```yaml
# equivalent zum CLI Befehl
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind Deployment
    name: nginx
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

##### Deployment löschen
Mit dem Befehl `kubectl delete deployment` werden Deployments gelöscht. Alle zugehörigen Pods und Ressourcen (Replicasets) werden ebenfalls gelöscht. Eine Skalierung auf 0 vor dem Löschen ist optional, aber Empfohlen. Mit der Options `--cascade=false` wird das Löschen abhängiger Ressourcen verhindert., mit `--grace-perid=<Sekunden>` wird eine Karenzzeit vor dem Löschen festgelegt.

##### Gesundheitscheck für Deployments
Eine Liveness Probe prüft, ob der Container läuft und reagiert. Es gibt diverse Arten von Probes: HTTP-Anfragen, TCP.-Socket-Verbindungen, Ausführen von Befehlen im Container. Die Prüfung kann mit verschiedenen Parametern konfiguriert werden: Häufigkeit (`periodSeconds`), Fehlerschwelle (`failureThreshold`) und Wartezeit (`initialDelaySeconds`).
Für professionelle Applikationen und Workloads sind `livenessProbes` unbedingt notwendig.

##### Verfügbarkeit von Deployments prüfen
`readinessProbe` prüft ob ein Container bereit ist, Anfragen zu verarbeiten. Bei einem Fehlschlage wird der Pod neu gestartet. Dabei gibt es verschieden Arten von Überprüfungen (HTTP-Anfragen, TCP-Socket-Verbindungen, Ausführen von Befehlen im Container) und fein granulare Konfigurationen: Häufigkeit (`periodSeconds`), Fehlerschwelle (`failureThreshold`) und Wartezeit (`initialDelaySeconds`) der Überprüfungen.
`livenessProbe`und `readinessProbe` werden häufig zusammen eingesetzt.

##### StatefulSets
https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
Ein zustand behaftetes Deployment.
Wird üblicherweise bei Datenbanken, verteilten Systemen oder Anwendungen die stabile Netzwerkidentitäten oder persistenten Speicher benötigen eingesetzt. Ein `StatefulSet` garantiert die geordnete Erstellung, Skalierung und Löschung von Pods. Jeder Pod hat eine eindeutige Identität und einen stabilen Netzwerknamen. `StatefulSet` unterstützt einen persistenten Speicher für jeden Pod.

Es müssen Abhängigkeiten definiert sein, `StorageClass` etc.
Definition und Bereitstellung sind analog zu "normalen" Deployments, durch die möglichen Konfigurationen sind `StatfulSet` besonders für Datenbanken oder ähnliches geeignet.

Beispielkonfiguration:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:latest
          ports:
            - containerPort: 27017
              name: mongo
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongodb-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: local-path
        resources:
          requests:
            storage: 1Gi
```

##### DaemonSets
https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
Deployments auf allen Nodes.
Werden üblicherweisen für Logging, Monitorong, Netzwerk-Plug-Ins oder Cluster-Agenten, d.h. die überall laufen sollen, eingesetzt.
Diese laufen auf jedem Node in einem Cluster, durch Node-Affinitäten können diese aber an bestimmte Nodes gebunden werden.
Ein `DaemonSet` erstellt automatisch neue Pods auf neuen Nodes und löscht sie auf entfernten Nodes.
Grundsätzlich gilt: **alle Pods eines `DaemonSet` laufen auf allen Nodes.**

##### Jobs
https://kubernetes.io/docs/concepts/workloads/controllers/job/
Aufgaben bezogene Jobs.
Werden meist genutzt um Aufgaben einmalig auszuführen, die außerhalb eines normalen Kontextes stattfinden, also Batch-Verarbeitungen, Datenanalyse oder einmalige Aufgaben.
Jobs erstellen Pods für genau eine Aufgabe und beenden den Pod nach der Ausführung wieder, ohne diese erneut zu starten, falls ein Fehler aufgetreten ist.

##### CronJobs
https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
Zeit gesteuerte Ausführung von Jobs.
Werden für wiederkehrende Aufgaben genutzt, z.B. Back-ups, Berichte, Datenbereinigung und andere regelmäßige Aufgaben.
`CronJobs` erstellen `Jobs` zu festgelegten Zeiten oder Intervallen und die Konfiguration basiert auf dem Linux-Cron-Format für Zeitplanungen.
Zusätzlich bieten `CronJobs` Wiederholungen und Fehlerbehandlungen bei fehlschlagenden Jobs an.

### 2025-04-19

#### Replicasets
https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/
Pods verfügen über keinen Mechanismus zur Skalierung.
Wenn Pods ausfallen oder gelöscht werden, werden sie nicht automatisch wiederhergestellt.
Pods bieten keine Funktionalitäten rund um unterbrechungsfreie (Zero-Downtime) Updates.

Replicasets stellen sicher, dass immer die gewünschte Anzahl an Pods läuft und sie bieten Funktionalitäten wie Skalierung und unterbrechungsfreie Updates an.
Replicasets werden über K8s, in aller Regel im Yaml-Format definiert, verwaltet.

Definition eines Replicasets:
```yaml
apiVersion: v1
kind: Replicaset
metadata:
	name: linkedin-replicaset
spec:
	selector:
		matchLabels:
			app: linkedin-pod
template:
	metadata:
		labels:
			app: linkedin-pod
	spec:
		containers:
		- name: linkedin-container
		  image: busybox
# ...
```
Das Replicaset sucht nach einer genauen Übereinstimmung zwischen `spec/selector/matchLabels` und `template/metadata/labels`
#### Labels und Annotationen
Labels und Annotationen definieren:
```yaml
apiVersion: v1
kind: Pod
metadata:
	name: linkedin-pod
	labels:
		app: linkedin
		environment: prod
	annotations:
		author: "Daniel Nagel"
		version: "1.0.0"
spec:
	# ...
```
##### Labels
https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
Labels sind Schlüssel/Wert-Paare. Die Schlüssel müssen eindeutig sein und dürfen maximal 63 Zeichen lang sein. Die Schlüssel können für Filterung/Auswahl genutzt werden.
Werden üblicherweise genutzt um Kubernetes-Objekte zu finden (mögliche Operatoren: `=`, `!=`, `in`, `notin`).
Das ermöglicht die flexible Zuordnung von Kubernetes-Objekten zueinander, was Labels essentiell für die Automatisierung und das Verwalten von Clustern macht.

Die Labels einer Node können festlegen ob ein Pod auf einem bestimmten Node laufen darf oder nicht:
```yaml
apiVersion: v1
kind: Pod
metadata:
	name: linkedin-pod
	labels:
		app: linkedin
		environment: prod
	nodeSelector:
		disktype: ssd
		architecture: arm
spec:
	# ...
```
##### Annotationen
https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/
Annotationen sind Schlüssel/Wert-Paare. Die Schlüssellänge ist unlimitiert und kann nicht zur Filterung/Auswahl genutzt werden. Es handelt sich bei Annotationen um reine Meta-Informationen.
#### Namensräume
https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
Namensräume (`namespaces`) trennen und organisieren Ressourcen für isolierte Umgebungen (Entwicklung, Test, Produktion).
Es gibt vordefinierte Standard-Namensräume, welche mit `kube` beginnen, die in Ruhe gelassen werden sollten.
Eine Übersicht aller Namensräume erhält man mit `kubectl get namespaces`.
https://kubernetes.io/docs/tasks/administer-cluster/namespaces/#creating-a-new-namespace
Einen neuen Namensraum erstellt man mit `kubectl create namespace`.
Auflisten der Pods in einem Namensraum mit `kubectl get pod -n kube-system`.
Grundsätzlich gilt, alle `kubectl` Befehle können auf einem bestimmten Namensraum ausgeführt werden `kubectl ... -n <namespace>`.

`kubectl` administriert nur die Pods im `default` Namensraum, wenn kein Namesraum angegeben wurde.
https://kubernetes.io/docs/tasks/administer-cluster/namespaces/#deleting-a-namespace
Mit `kubectl delete namespace` kann ein gesamter Namensraum mit allen Ressourcen gelöscht werden. Dieser Vorgang kann etwas Zeit in Anspruch nehmen.

Der Standard Namensraum kann in der Konfiguration festgelegt werden. Dies erfolgt mit dem Befehl `kubectl config set-context --current --namespace default`.
#### Node-Affinitäten
https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors
Node-Affinitäten sind Regeln für die Pod-Platzierung auf Nodes. Basieren auf dem Vorhandesein oder der Abwesenheit bestimmter Labels auf Nodes.
Definieren weiche ("bevorzugte", `nodeAffinity`) und harte ("ausschließlich" `nodeSelector`) Regeln.
Node-Affinitäten erzwingen Übereinstimmungen mit Node-Labels.
Erst bei einer Übereinstimmung wir der Pod ausgerollt.
Label zu Node hinzufügen: `kubectl label node`
##### nodeSelector
Einfache Definition durch harte Regeln, da eine genau Übereinstimmung nötig ist.
##### nodeAffinity
Größere Flexibilität, da harte und weiche Regeln durch Listen und Ausschlusskirterien möglich sind (`In`, `NotIn`, `Exists`, ...)

Node-Affinitäten kommen in echten Clustern in Frage, in Single-Node-Clustern macht das wenig Sinn. Das ist zwar gut zu Wissen, aber für meine Fälle erstmal irrelevant.

### 2025-04-18

#### Sidecars

https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/

Ein zusätzlicher Container welcher mit dem Haupt-Container zusammen in einem Pod läuft.
Dieser bietet zusätzlich Funktionen für den Haupt-Container ohne diesen zu verändern. Haupt- und Sidecar-Container teilen sich die Ressourcen, etwa Netzwerk oder Speicher.
Sidecars sind zusätzliche Einträge in der Konfiguration, so wie `initContainers`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: linkedin-sidecar
spec:
  containers:
    - name: linkedin-container
      image: busybox:latest
      command: ["sh", "-c", "while true; do echo $(date) >> /data/time.txt; sleep 5; done"]
      volumeMounts:
        - name: data
          mountPath: /data
    - image: busybox:latest
      name: sidecar
      command: ["sh", "-c", "tail -f /data/time.txt"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      emptyDir: {}
```

Ein Sidecar hat den selben Lebenszyklus wie der Hauptcontainer, d.h. der Sidecar Container wird mit dem Hauptcontainer gestartet und beendet.

Mir ist aufgefallen das Konfiguration nicht erneut angewendet werden können (`apply`), der betroffene `Pod` muss zuvor gelöscht werden.

Bei der Verwaltung von Sidecars ist zu beachten, dass diese bei der Verwaltung explizit angegeben werden müssen.

```bash
# Logs des Haupt-Containers
k logs linkedin-sidecar
# Logs des Sidecars (Der Name des Containers ist sidecar)
k logs linkedin-sidecar -c sidecar
```

#### Requests und Limits

Requests und Limits definieren minimal und maximale CPI- und Speichervorgaben. Diese werden von K8s zur Zuordnung von Pods zu Nodes genutzt. Die Angabe schützt vor übermäßiger Nutzung und Engpässen und stellt die Cluster-Stabilität sicher.

Requests und Limits sind extrem wichtig, nur so kann k8s die Cluster-Stabilität sicherstellen.
Produktiv müssen Requests und Limits vorhanden sein, sonst können Applikationen nicht nachhaltig betrieben werden!
##### Requests

Requests definieren Untergrenzen, d.h. minimale Ressourcenanforderungen der Pods, somit wird die Verfügbarkeit sichergestellt. K8s nutzt diese Informationen um sicherzustellen auf welchen Nodes die Pods laufen können und ob diese überhaupt noch laufen können (Scheduling und Pod-Zuordnung).

##### Limits

Limits definieren Obergrenzen, d.h. diese werden dazu verwendet damit K8s Pods überwachen kann. Diese legen eine maximale Ressourcenbegrenzung fest, vermeiden Übernutzung und sorgen dafür das K8s die Pods bei Überschreitung beendet. Die Cluster-Stabilität hat immer Vorrang gegenüber der Gesundheit einzelner Pods.

#### Busybox Image

Das **BusyBox**-Image ist ein leichtgewichtiges, minimalistisches Linux-Image, das eine Sammlung von Unix-Utilities in einer einzigen ausführbaren Datei kombiniert. Es wird häufig in Container-Umgebungen verwendet, da es extrem klein ist (wenige Megabyte) und dennoch grundlegende Funktionen für Shell-Skripte und Systemadministration bereitstellt.

BusyBox enthält viele grundlegende Unix-Utilities wie `ls`, `cp`, `mv`, `cat`, `echo`, `grep`, `vi`, `wget` und viele mehr. Es wird häufig als Basis-Image für Container verwendet, die keine vollständige Linux-Distribution benötigen. 

Es könnte verwendet werden, um einfache Aufgaben wie das Ausführen von Shell-Skripten, Debugging oder das Bereitstellen von Testumgebungen zu erledigen.
Zum Beispiel könnte es für einen Sidecar-Container verwendet werden, der Logs sammelt oder einfache Netzwerkoperationen ausführt.
### 2025-04-17
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