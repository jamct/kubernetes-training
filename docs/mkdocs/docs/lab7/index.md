# :fa-flask: Lab 7: Entwickler-Freuden und Debugging-Tipps

Zum Abschluss ein paar nützliche Hilfsmittel, die Ihnen den Alltag erleichtern werden (in der Reihenfolge ihres Vorkommens in der Veranstaltung).

## Liveness-Probes

Beim Updaten von Setups soll die Ausfallzeit natürlich so gering wie möglich sein. Das können Sie erreichen, indem Sie `ReadinessProbes` setzen. Kubernetes pürft dann, ob der Container hochgefahren ist.

`LivenessProbes` prüfen während des Betriebs, ob der Container richtig arbeitet. So sieht die Einrichtung innerhalb eines Deployments aus:

```
  - name: beispiel
    image: mein-webserver
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3

    readinessProbe:
    httpGet:
        port: 8080
        path: /readiness
        scheme: HTTP
    initialDelaySeconds: 3
    periodSeconds: 3
    successThreshold: 1 
    failureThreshold: 1 
    timeoutSeconds: 1
```

Als Entwickler sollten Sie passende Endpunkte an Ihre Dienste anbauen. Das erleichtert dem Ops-Kollegen die Arbeit.

## Aus dem Cluster ins Cluster

Manchmal möchten Sie einen Container entwickeln, der sich Informationen aus dem Cluster bezieht. Für die Zugriffe braucht der Container einen ServiceAccount mit den passenden Rechten (siehe Lab 5).

Kubernetes legt das Zugriffstoken immer ab unter

```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

Der API-Server ist intern erreichbar unter `https://kubernetes.default.svc`.

Weitere Infos und Bibliotheken in der [Kubernetes-Doku](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/)

## Besuch im Container

Manchmal wollen Sie wissen, wie es im Container aussieht. Wie unter Docker auch, gibt es dafür den Subcommand `exec`:

Zum Beispiel:

```
kubectl exec -it test-234234 sh
```

## Besuche aus der Ferne: Telepresence

Manchmal kommt man mit dem lokalen Experimentiercluster nicht weiter, das man zum Beispiel mit Docker Desktop oder Minicube erzeugt. Dann würde man am liebsten einen lokalen Container in einem richtigen Cluster einpflanzen, um einen Fehler an Ort und Stelle zu finden. Genau das funktioniert mit [Telepresence](https://www.telepresence.io). Die Software hat ihre Tücken und wird aktiv weiterentwickelt. Das Prinzip: Auf der Entwicklermaschine wird ein Container gestartet (Docker muss installiert sein). Mit dem Kommandozeilenbefehle `kubectl` entsteht ein Platzhalter-Container im Cluster, der einen bestehenden Pod ersetzt. Dieser Platzhalter stellt die Verbindung zu Ihrem lokalen Pod her. Ihr lokaler Container läuft jetzt, als wäre er im Cluster.

## Private Registrys

Nicht immer sind alle Images öffentlich. Dann müssen Sie ein Secret im Cluster ablegen, das ein Token für die Registry Ihrer Wahl enthält.

Melden Sie sich zunächst auf der lokalen Maschine mit `docker login` an der Registry an. Schauen Sie dann in die Datei `~/.docker/config.json`

Schreiben Sie den Inhalt als Secret ins Cluster:

```
kubectl create secret generic my-docker-secret \
    --from-file=.dockerconfigjson=~/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson
```

In der Definition eines Pods übergeben Sie das Secret als `imagePullSecrets`. Es kann dann für alle Container des Pods verwendet werden:

```
  containers:
  - name: mein-server
    image: private-registry/privat:latest
  imagePullSecrets:
  - name: my-docker-secret
```
