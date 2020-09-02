# :fa-flask: Lab 6: Die Nextcloud-Instanz

Im letzten großen Block sehen Sie, wie eine Nextcloud-Instanz ins Cluster gelangt. Auch dabei gibt es neue Konzepte zu entdecken.

## Secrets: Aufbewahrung für Geheimnisse

In Docker-Umgebungen sieht man oft Umgebungsvariablen wie

```
environment:
  SECRET_TOKEN=ü0p39p4958ü2ßw0p344895ßü0234895
```

zusammen mit dem Rest des Deployments im GitHub-Repo. Kubernetes erlaubt es uns, solche Geheimnisse separat als Objekt anzulegen und per RBAC zu steuern, wer sie lesen darf. Ein Secret wird erstmal als Objekt angelegt und benannt:

```
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  password: "veryVerySecret"
  rootpassword: "muchMoreSecret!"
```

So ein Secret kann mehrere Inhalte haben (der Inhalt ist auch nur YAML). Dieses Secret nutzen wir gleich für die Datenbank.

## Die Datenbank

Beim Ausrollen der Datenbank gibt es wenig Neues zu entdecken. Es gibt zunächst ein klassisches Deployment.

Neues finden Sie in den Umgebungsvariablen. Hier gibt es einen Block, der das oben gebaute Secret benutzt:

```
- name: MYSQL_ROOT_PASSWORD
    valueFrom:
    secretKeyRef:
        name: db-secret
        key: rootpassword
```

Die Datenbank bekommt einen Service. Damit können andere Pods im Cluster sie unter dem Namen `nc-db.default.svc.cluster.local` (die Endung `.default.svc.cluster.local` kann man weglassen. Der DNS-Server kümmert sich schon drum).


```
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: nc-db
  labels:
    app: nc-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nc-db
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nc-db
    spec:
      volumes:
       - name: nc-db-vol
         persistentVolumeClaim:
           claimName: pvc-nc-db
      containers:
        - name: database
          image: mariadb
          imagePullPolicy: Always
          volumeMounts:
           - name: nc-db-vol
             mountPath: "/var/lib/mysql"
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: rootpassword
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
            - name: MYSQL_DATABASE
              value: "nextcloud"
            - name: MYSQL_USER
              value: "nextcloud"
            - name: MYSQL_INITDB_SKIP_TZINFO
              value: "1"
---
apiVersion: v1
kind: Service
metadata:
  name: nc-db
  namespace: default
spec:
  ports:
    - protocol: TCP
      name: sql
      port: 3306
  selector:
    app: nc-db
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nc-db
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi
```

## Die Nextcloud selbst

Jetzt die Nextcloud selbst. Auch sie bekommt das Kennwort für die Datenbank. Außerdem wird ihr der Name des Datenbank-Service übergeben. 

```
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: nc
  labels:
    app: nc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nc
  template:
    metadata:
      labels:
        app: nc
    spec:
      volumes:
       - name: nc-data-vol
         persistentVolumeClaim:
           claimName: pvc-nc-data
      containers:
       - name: web
         image: nextcloud:stable
         imagePullPolicy: Always
         volumeMounts:
           - name: nc-data-vol
             mountPath: "/var/www/html"
         ports:
          - name: web
            containerPort: 80
         env:
          - name: MYSQL_HOST
            value: nc-db
          - name: NEXTCLOUD_TRUSTED_DOMAINS
            value: cloud.0.liefer.it
          - name: MYSQL_DATABASE
            value: nextcloud
          - name: MYSQL_USER
            value: nextcloud
          - name: MYSQL_PASSWORD
            valueFrom:
              secretKeyRef:
                name: db-secret
                key: password
          - name: NEXTCLOUD_ADMIN_USER
            value: "admin"
          - name: NEXTCLOUD_ADMIN_PASSWORD
            value: "admin"
          - name: APACHE_DISABLE_REWRITE_IP
            value: "1"
          - name: TRUSTED_PROXIES
            value: "10.0.0.0/8"
---
apiVersion: v1
kind: Service
metadata:
  name: nc
  namespace: default
spec:
  ports:
    - protocol: TCP
      name: web
      port: 80
  selector:
    app: nc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nc-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 20Gi
```

## Die IngressRoute

Zum Schluss bekommt die Nextcloud eine IngressRoute. Die Middlewares sind kein Konzept von Kubernetes, sondern von Træfik. Mit Middlewares beeinflussen Sie den Anfragestrom. Mehr darüber in der Doku.

```
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: nextcloud-service-discovery
spec:
  redirectRegex:
    permanent: true
    regex: /.well-known/(card|cal)dav
    replacement: /remote.php/dav/

---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: nextcloud-headers
spec:
  headers:
    stsSeconds: 15552000
    stsIncludeSubdomains: true

---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressroute-nc
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`cloud.0.liefer.it`)
      kind: Rule
      services:
        - name: nc
          port: 80
      middlewares:
        - name: nextcloud-service-discovery
        - name: nextcloud-headers
  tls:
    certResolver: default
```