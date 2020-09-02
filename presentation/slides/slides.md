 ## Herzlich Willkommen

![ ](https://www.heise-events.de/uploads/WQ8OofZU/767x0_2500x0/Kubernetes_Docker_2000x500.jpg)

1. und 2.9.2020
---

## Ihr Dozent

* Jan Mahn
* seit 2017 c't-Redakteur im Ressort Systeme und Sicherheit
* zuständig für Server-Systeme, Container, Software-Entwicklung
* Docker-Nutzer seit 2017
* Kubernetes-Nutzer seit 2018

---

## Das Ziel für heute und morgen

* ein eigenes Cluster einrichten
* Dienste im Cluster betreiben
* nützliche Grundfunktionen kennenlernen

---

## Warum wir Container nutzen

Es geht um:
* Flexibilität
* Ersetzbare Server
* Weniger Ärger mit Abhängigkeiten

--- 

## Nutztiere und Haustiere

"In der Container-Welt sind Container nur Nutzvieh. Sie sind ersetzbar! Stirbt einer, übernimmt der nächste. Kein Grund zur Trauer."

---

## Der typische Docker-Host

* enthält eine Sammlung an Docker-Compose-Dateien 
* auf dem Docker-Host wird mit `docker` und `docker-compose gearbeitet` 
* nach einiger Zeit baut man wieder eine Beziehung zum Server auf und pflegt ihn mehr, als man eigentlich wollte

---

## Die Evolution der Technik

* Virtualisierung löst all unsere Probleme mit Servern <!-- .element: class="fragment" data-fragment-index="1" -->
* Docker löst all unsere Probleme mit Virtualisierung <!-- .element: class="fragment" data-fragment-index="2" -->
* Kubernetes löst all unsere Probleme mit Docker <!-- .element: class="fragment" data-fragment-index="3" -->
* ... <!-- .element: class="fragment" data-fragment-index="4" -->

---

## Wem gehört Kubernetes?

* erfunden von Google, ab 2014
* Open-Source seit 1.7.2015
* gespendet an die CNCF, Teil der Linux Foundation
* 2820 Contributors bei GitHub
* Mitarbeit durch alle großen und viele kleine Cloud-Unternehmen

---

![ ](./slides/stats.png)

---

## Strategien zum Lernen von K8S

---

## Kubernetes: Containern weiter gedacht

Mit einem Kubernetes-Cluster:

* beschreiben Sie Ihre gewünschte Infrastruktur, meist in YAML.
* konfigurieren Sie Details, die in `docker-compose` fehlen.
* lassen Sie Container auf Funktion prüfen und ersetzen. 
* skalieren Sie Dienste, wenn die Last höher wird.
* verteilen Sie die Last auf viele Maschinen.

---

## Was Kubernetes anders macht

* Läuft auf mehreren Hosts (max. 5000 pro Cluster)
* ist auf Skalierbarkeit ausgelegt
* Arbeitet (meist) deklarativ, nicht imperativ

--- 

## Docker arbeitet imperativ

"Hey Docker, starte einen Container"

```
docker run ...
```

"Hey Docker, lösche ihn wieder"

```
docker stop ...
```

Auch docker-compose ist imperativ

---

## Kubernetes-Admins definieren Zustände

"Hey Cluster: So soll die Umgebung aussehen. Kümmere dich drum."

---
## Aufbau eines Clusters

![ ](./slides/cluster.png)

---

## Lab 1: Das eigene Cluster

Alle Workshop-Inhalte finden Sie unter [docs.liefer.it](https://docs.liefer.it)

![ ](https://heise.cloudimg.io/width/900/q65.png-lossy-65.webp-lossy-65.foil1/_www-heise-de_/select/ct/2016/5/1456733697045992/contentimages/image-145552165478819.jpg)

---

## Begriffskunde

* **Nodes** sind Server mit installiertem Kubernetes, zu einem Cluster verbunden
* **Pods** sind die kleinste verteilbare Einheit. Sie können aus mehreren **Containern** bestehen.
* **Services** sind die Weiterentwicklung von `ports` in einer Docker-Umgebung. Sie erlauben elegantes Loadbalancing

---

## Was sind Pods?
![ ](./slides/node.png)

---

## Einen Container ausrollen

Mit docker-compose
```
services:
  whoami:
    image: containous/whoami
    ports:
      - 80:80
```

---

Mit Kubernetes:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - image: containous/whoami
        imagePullPolicy: Always
        name: whoami
        ports:
        - containerPort: 80
```
---
... und weiter geht es:

```
apiVersion: v1
kind: Service
metadata:
  name: whoami-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: whoami
  ports:
  - protocol: TCP
    nodePort: 30000
    port: 8000
    targetPort: 80
```
---

## Lab 3: Der erste Dienst im Cluster

Alle Workshop-Inhalte finden Sie unter [docs.liefer.it](https://docs.liefer.it)

---

## Ihre Fragen zum ersten Tag

Am zweiten Tag routen wir eingehenden Verkehr, schauen auf Logs und veröffentlichen eine komplexere Zusammenstellung mit persistentem Speicher.

Sollten im Laufe des Nachmittags Fragen gekommen sein: jam@ct.de