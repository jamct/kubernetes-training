# :fa-flask: Lab 8: Helm – ein Verpackungskünstler

Mit Helm verpacken Sie Ihr Kubernetes-Projekt, um es auf mehreren Maschinen auszurollen.

```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

## Fertige Charts

Mit Helm laden Sie fertige Charts aus dem Helm Hub:

https://hub.helm.sh

## Eigene Charts

Helm hat eine Go-Template-Engine an Bord. Alles in `{{ }}` wird als Variable ersetzt und interpretiert. Ein Beispiel für ein solches Paket finden Sie im Projekt [Team-Container](https://github.com/ct-open-source/team-container).