# :fa-flask: Lab 1: Das erste Cluster

In diesem Lab entsteht ein kleines Cluster, das auf zwei Maschinen läuft.



Ein Cluster besteht aus einem oder mehreren `nodes`. Zwei Rollen gibt es in einem Cluster:

* `master` verwalten das System und stellen u.a. das API bereit
* `worker` betreiben die Arbeitslasten, also die Container

Ein `node` kann beide Rollen ausfüllen. In großen Systemen sollte man die Rollen trennen. Im Hintergrund sprechen die Master mit einer Datenbank, zum Beispiel der Key-Value-Datenbank etcd, die redundant auf allen Mastern laufen kann (oder auch auf Maschinen außerhalb).

Die Mindestanzahl an Nodes hängt von der Kubernetes-Distribution und der verwendeten Datenbank ab. Für den Workshop kommt die kleine Distro K3S zum Einsatz, die von Rancher erfunden wurde. Rancher wiederum gehört neuerdings zu SuSe.

K3S läuft zur Not auch auf einer Maschine und ist eine ideale Lösung für alle, die in das Thema einsteigen möchten. Beschaffen Sie sich 1 bis 3 Linux-Maschinen bei einem Hoster des Vertrauens und stecken Sie diese in ein internes Netzwerk. Wenn Sie eine kostengünstigere Lösung suchen: K3S läuft auch auf ARM-Prozessoren. Drei Raspis sind ein brauchbares Übungs-Cluster!

Für den Workshop haben Sie zwei x64-Maschinen erhalten. Loggen Sie sich am besten mit zwei Fenstern nebeneinander ein.

## Netzwerk vorbereiten

Finden Sie nacheinander die IP-Adresse im internen Netz heraus:

```
hostname -I
```

Die Ausgabe sieht etwa so aus:

```
159.69.0.189 10.20.0.3 10.42.0.0 10.42.0.1 2a01:4f8:c2c:3d59::1
```

Die erste Adresse ist die externe Adresse, die zweite (aus dem 10er-Netz, zum Beispiel 10.20.0.3) ist die Gesuchte Adresse des lokalen Netzes Ihrer beiden Maschinen. Wiederholen Sie den Schritt für die zweite Maschine.


## K3S beschaffen

Die Installation von K3S haben die Entwickler in ein Installations-Skript verpackt. Per Umgebungsvariable übergeben Sie Einstellungen. Auf der ersten Maschine (zum Beispiel demo-00-1) soll das Cluster initialisiert werden.

Vor der Installation müssen Sie sich ein Token ausdenken, das alle Kubernetes-Nodes kennen müssen. Ändern Sie das Token in der folgenden Zeile ab und führen Sie den Einzeiler auf der ersten Maschine aus:

```
curl -sfL https://get.k3s.io | K3S_TOKEN=verySecretToken INSTALL_K3S_EXEC="--disable traefik --write-kubeconfig-mode 644 --cluster-init" sh -
```

Im Hintergrund wird K3S heruntergeladen. Nach etwa 3 Minuten dürfte der Assistent fertig sein. Drei Anmerkungen zu den Parametern:

* `--disable-traefik` deaktivert den HTTP-Router Træfik. Diesen installieren Sie später im Cluster und lernen an diesem Beispiel viele Grundfunktionen kennen.
* `write-kubeconfig-mode 644` erlaubt Ihnen das lokale Arbeiten mit der Kommandozeile `kubectl`.
* `--cluster-init` legt das Cluster an.

Auf der zweiten Maschine müssen Sie ebenfalls K3S herunterladen und zwei Werte übergeben:

* das Token von Server 1
* die IP-Adresse von Server 1

Ändern Sie diese beiden Angaben in der folgenden Zeile:

```
curl -sfL https://get.k3s.io | K3S_TOKEN=verySecretToken INSTALL_K3S_EXEC="--disable traefik --server https://10.20.0.3:6443" sh -
```

Nach weiteren drei Minuten ist Ihr Cluster einsatzbereit.

## Zustand prüfen

Mit dem Kommandozeilenprogramm `kubectl` prüfen Sie, ob die Nodes zusammenarbeiten:

```
kubectl get nodes
```

Auf beiden Maschinen sollten Sie eine Ausgabe wie die Folgende sehen:

```
NAME       STATUS   ROLES    AGE     VERSION
demo-0-1   Ready    master   4m14s   v1.18.8+k3s1
demo-0-2   Ready    master   18s     v1.18.8+k3s1
```

Im [zweiten Lab](../lab2) lernen Sie Ihren neuen Freund `kubectl` näher kennen.