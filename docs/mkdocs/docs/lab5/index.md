# :fa-flask: Lab 5: Ein Router für den eingehenden Verkehr

Jetzt geht es ans Eingemachte. Im Demo-Cluster soll eine Nextcloud-Instanz laufen. Am Beispiel kann man viele Bausteine des üblichen Cluster-Alltags ausprobieren. Folgende Anforderungen gibt es:

* die Seite soll mit TLS und gültigem Zertifikat laufen (Let's Encrypt, mit Auto-Renewal)
* die Daten sollen persistent sein (Daten landen auf der lokalen Platte)
* die Zugangsdaten zur Datenbank sollen nicht in einer Konfigurationsdatei liegen

## Der Router für eingehenden Verkehr

Als Router für den eingehenden Verkehr und als Beschaffer von LE-Zertifikaten soll Traefik zum Einsatz kommen.

Legen Sie am besten einen Ordner lokal an und erstellen Sie dort nach Belieben Dateien.

### 1. CRDs

Gleich zu Beginn ein neues Konzept, das etwas ungewohnt erscheinen mag, aber extrem mächtig ist. Mit CRDs (Custom Resource Definitions) bohren Sie das Schema des Kubernetes-APIs auf. Viele Anwendungen nutzen das, um Konfigurationen im Cluster abzulegen. So auch Træfik. Auch die Hüllen für CRDs kann man als YAML formulieren und mit `kubectl`veröffentlichen:

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsstores.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSStore
    plural: tlsstores
    singular: tlsstore
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressrouteudps.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteUDP
    plural: ingressrouteudps
    singular: ingressrouteudp
  scope: Namespaced
```

Wenn Sie diese CRDs veröffentlicht haben, lernt Ihre `kubectl` automatisch neue Begriffe (ganz ohne Update). Der Grund: In den CRDs werden Singular und Plural für `kubectl` bereits definiert. Probieren Sie etwa:

```
kubectl get ingressroutes
```

## Roles und Permissions

Noch ein verwirrendes Konzept, das zunächst übermäßig komplex wirkt. In der Docker-Welt gibt es ein wiederkehrendes Muster: Immer, wenn ein Container selbst etwas mit dem Docker-Host anstellen soll, bekommt er diesen als Volume. Das sieht dann in Docker so aus:

```
/var/run/docker.sock:/var/run/docker.sock
```

Docker kennt hier nur "Alles oder Nichts!". Wenn ein Container den Docker-Socket bekommt, kann er damit alles anstellen, was Sie auch als Admin mit `docker` können. Einzige Einschränkung: Hängen Sie `:ro` an, kann der Container nur lesen. Ein bisschen schreiben geht aber nicht.

Kubernetes ist – nicht überraschend – viel flexibler und hat ein ausgefeiltes RBAC-System. Im Folgenden wird für Træfik, der später andere Pods erkennen und deren Metadata auslesen soll, eine eigene Rolle geschaffen. Diese bekommt fein aufgeschlüsselt, was sie alles darf.

Im zweiten Teil wird ein `ServiceAccount` erstellt. Im dritten Teil wird der `ServiceAccount` mit der Rolle verknüpft.

Kopieren Sie alles in eine Datei und senden Sie es per `apply` ans Cluster.

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
      - ingressroutes
      - traefikservices
      - ingressroutetcps
      - ingressrouteudps
      - tlsoptions
      - tlsstores
    verbs:
      - get
      - list
      - watch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: default
  name: traefik-ingress-controller
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: default
```

### Volumes

Gleich noch ein neues Konzept: Endlich lernen Sie eine Möglichkeit kennen, persistente Daten zu speichern. Schon mal vorab: Die Welt wäre viel einfacher, gäbe es nur Stateless-Anwendungen!

Mit einem einfachen

```
volumes:
 - local:/pfad/im/container
```

ist es nicht getan. Stattdessen müssen Sie sich zuallererst entscheiden, wo die Daten am Ende landen sollen – Sie brauchen eine sogenannte `storageClass`. K3S bringt eine ganz einfache mit. Sie heißt `local-path` und legt die Dateien auf den Filesystems der Nodes ab. Einen Abgleich gibt es nicht.

Für unsere Demo reicht das aus, in Produktion werden Sie sicher auf eine verteilte Lösung setzen wollen. Zum Glück gibt es viele Möglichkeiten, die Sie für Ihre konkrete Anwendung mal ausprobieren sollten. Folgende Anlaufstellen empfehlen wir:

* [Die Kubernetes-Doku (immer eine gute Adresse, oft sehr detailliert...)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
* [Das Projekt Longhorn (auch von Rancher)](https://longhorn.io)

Gute Ansätze: Wenn Sie irgendwo bereits NFS betreiben, gibt es eine `StorageClass` eingebaut. Longhorn ist eine einfache Lösung. Longorn wird selbst per `kubectl` verteilt.

In jedem Fall brauchen Sie zunächst einen PVC – hat nichts mit Kunststoffen zu tun. PVC bedeutet `PersistentVolumeClaim`. Das ist die Vorbestellung von Speicherplatz. Der PVC enthält unter anderem die `storageClass`.

Træfik braucht etwas Platz für das Zertifikat:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: traefik-cert
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 128Mi
```

Speichern und deployen. Mit `kubectl get pvc` sollten Sie das Ergebnis sehen.

!!! note "Local-Path"
    In diesem Beispiel kommt Local-Path zum Einsatz. Diese `StorageClass` hat Rancher für K3S eingebaut. Es ist die minimale Implementierung einer `StorageClass`, die man am ehesten mit Volumes unter Docker vergleichen kann. Wunder sind davon nicht zu erwarten. Auf Multi-Node-Clustern ist Local-Path keine gute Idee. Er sorgt einfach dafür, dass ein Pod immer auf dem Node gescheduled wird, auf dem die Daten lokal liegen. Wenn dieser Node ausfällt, fehlen die Daten auf anderen Nodes. Außerhalb von Klein- und Testumgebungen sollten Sie eine andere `StorageClass`nutzen.

### Deployments

Jetzt ist es an der Zeit, Træfik selbst auszurollen. Das Deployment sollte Ihnen vertraut vorkommen. Neu ist vor allem das Einbinden der Volumes, das anders als in Docker einige Zeilen mehr kostet.

Erst wird das Volume auf Basis eines PVC für den gesamten Pod bekannt gemacht und mit einem Namen (hier `cert-vol`) versehen. Am konkreten Container wird dann der Ziel-Ordner angegeben: `moutPath: /data`.

```
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: traefik
  labels:
    app: traefik
spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      volumes:
       - name: cert-vol
         persistentVolumeClaim:
           claimName: traefik-cert
      containers:
        - name: traefik
          image: traefik:v2.2
          imagePullPolicy: Always
          volumeMounts:
           - name: cert-vol
             mountPath: "/data"
          args:
            - --api.insecure
            - --accesslog
            - --entrypoints.web.Address=:80
            - --entrypoints.websecure.Address=:443
            - --providers.kubernetescrd
            - --certificatesresolvers.default.acme.tlschallenge
            - --certificatesresolvers.default.acme.email=jam@ct.de
            - --certificatesresolvers.default.acme.storage=/data/acme.json
          ports:
            - name: web
              containerPort: 80
            - name: websecure
              containerPort: 443
---
# Servive for Ingress-Traefik
apiVersion: v1
kind: Service
metadata:
  name: traefik
spec:
  ports:
    - protocol: TCP
      name: web
      port: 80
      targetPort: 80
    - protocol: TCP
      name: websecure
      port: 443
      targetPort: 443
  selector:
    app: traefik
  type: LoadBalancer
```

Træfik bekommt außerdem einen Service vom ungewöhnlichen Typ `LoadBalancer`. Damit ist er bereit für den Betrieb als Router für eingehenden Verkehr. Er blockiert die Ports 80 und 443.

### Ein kleiner Server

Das folgende Deployment ist ein kleiner Kunstgriff. Es sorgt dafür, dass auch ohne weitere Dienste ein Zertifikat beschafft wird. Damit können Sie die Funktion testen.

Ausgerollt wird ein einfacher NGINX mit einem Service.

```
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: landingpage
  labels:
    app: landingpage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: landingpage
  template:
    metadata:
      labels:
        app: landingpage
    spec:
      containers:
        - name: landingpage
          image: nginx:alpine
          imagePullPolicy: Always
          ports:
            - name: web
              containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: landingpage
spec:
  ports:
    - protocol: TCP
      name: web
      port: 80
  selector:
    app: landingpage
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressroute-landingpage
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`www.0.example.org`) && Path(`/`)
      kind: Rule
      priority: 1
      services:
        - name: landingpage
          port: 80
  tls:
    certResolver: default
```

Der letzte Abschnitt ist Træfik-spezifisch: Die `IngressRoute` ist kein Standard-Objekt. Wir haben es am Anfang per CRD ans Kubernetes-API angebaut und nutzen es jetzt.

Diese Objekte liest Træfik zur Laufzeit ein und arbeitet damit – er merkt sich alle Routen und beschafft direkt Zertifikate dafür.

Am Ende dieses Labs sollten Sie unter https://www.0.example.org (Ihre Zahl einsetzen) einen Nginx mit Zertifikat sehen! Um die Erneuerung in drei Monaten kümmert sich Træfik.