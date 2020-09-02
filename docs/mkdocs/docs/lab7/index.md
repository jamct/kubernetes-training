# :fa-flask: Lab 7: Entwickler-Freuden und Debugging-Tipps


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
