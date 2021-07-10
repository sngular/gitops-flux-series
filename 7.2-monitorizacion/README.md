# 7.2 Monitorización

**TODO**

- [X] Carga de dashboards en Grafana [Manu]
- [ ] Suspender release [Manu]
- [x] Enseñar los logs con Loki (controllers) [Enrique]
  - [x] Query
  - [x] Live
- [ ] Desplegar algún servicio más?

A lo largo de esta guía se desplegará el stack de monitorización Loki + Grafana + Prometheus con el objetivo de mostrar la importancia de la observabilidad en los despliegues realizados con Flux. Además se pondrán en marcha algunos servicios para obtener métricas y logs, y demostrar la importancia de tener visibilidad sobre las actividades y eventos que ocurren en el cluster.

Vídeo de la explicación y la demo completa en este [enlace](https://www.youtube.com/watch?v=9IwXibOfSDk&list=PLuQL-CB_D1E7gRzUGlchvvmGDF1rIiWkj&index=9).

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >=0.16.0

## Exportar token de GitHub

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

## Instalar Flux en el cluster

Utilice el comando `bootstrap` para instalar los componentes de flux en el cluster, crear el repositorio en GitHub y mucho más:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=gitops-flux-series-demo \
  --branch=main \
  --private=false \
  --path=./clusters/demo
```

<details>
  <summary>Resultado</summary>

  ```bash
  ► connecting to github.com
  ✔ repository "https://github.com/sngular/gitops-flux-series-demo" created
  ► cloning branch "main" from Git repository "https://github.com/sngular/gitops-flux-series-demo.git"
  ✔ cloned repository
  ► generating component manifests
  ✔ generated component manifests
  ✔ committed sync manifests to "main" ("f20fb16201be4cedc86860139c4c30a7a5569bf3")
  ► pushing component manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► installing components in "flux-system" namespace
  ✔ installed components
  ✔ reconciled components
  ► determining if source secret "flux-system/flux-system" exists
  ► generating source secret
  ✔ public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC42KfDLo5DDDJU+KcLtT155hVQ3Gtd/IQLO2RRqshtRcnGmNebupSzea9CRi2sEzk+cNStXYpci0DWXY7joRnInMg+K/YwPYQGDfL373UNOi7pW6KqnlPmgxvqKXRHIh2/N4PWm+lG43Iq625xHKF1ITzEHPrdRULKB1uF1qHHOJFDTCJKPJrkZBrBspkJc4O/eKzloEjXuBlFwoWm/YvFo04kk3MRqKGGcOB/euxN5xeHgtq2nIS8m1qdJxHvkSA2zgVw3URYWEX+x5qz2zsM9w7Kj9TghmrquICnGkpF6Q7OcDh1MmX+1mrTjkvW//Nlua2x91y/4LVpsWAJDEHL
  ✔ configured deploy key "flux-system-main-flux-system-./clusters/demo" for "https://github.com/sngular/gitops-flux-series-demo"
  ► applying source secret "flux-system/flux-system"
  ✔ reconciled source secret
  ► generating sync manifests
  ✔ generated sync manifests
  ✔ committed sync manifests to "main" ("53202cc8bd759a3e32e6dcc8e8c9b5968c7112e2")
  ► pushing sync manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► applying sync manifests
  ✔ reconciled sync configuration
  ◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
  ✔ Kustomization reconciled successfully
  ► confirming components are healthy
  ✔ source-controller: deployment ready
  ✔ kustomize-controller: deployment ready
  ✔ helm-controller: deployment ready
  ✔ notification-controller: deployment ready
  ✔ all components are healthy
  ```
</details>

Comprobar que los componentes han sido instalados:

```bash
  kubectl get pods --namespace flux-system
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                                       READY   STATUS    RESTARTS   AGE
  source-controller-85fb864746-4x4s2         1/1     Running   0          65s
  helm-controller-85bfd4959d-lsshl           1/1     Running   0          66s
  notification-controller-5c4d48f476-qltpw   1/1     Running   0          65s
  kustomize-controller-6977b8cdd4-qq482      1/1     Running   0          66s
  ```
</details>

## Clonar repositorio creado

```bash
{
  git clone git@github.com:$GITHUB_USER/gitops-flux-series-demo.git
  cd gitops-flux-series-demo
}
```

## Crear recursos

Crear los directorios necesarios:

```bash
mkdir -p ./clusters/demo/{sources,monitoring/dashboards,gitops-series}
```

Crear los namespaces:

```bash
{
cat <<EOF > ./clusters/demo/gitops-series/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-series
EOF

cat <<EOF > ./clusters/demo/monitoring/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
EOF
}
```

Crear las fuentes:

```bash
{
flux create source helm grafana \
  --url=https://grafana.github.io/helm-charts \
  --interval=5m \
  --namespace=flux-system \
  --export > clusters/demo/sources/grafana-helmrepository.yaml

flux create source helm sngular \
  --url=https://sngular.github.io/gitops-helmrepository/ \
  --interval=5m \
  --namespace=flux-system \
  --export > clusters/demo/sources/sngular-helmrepository.yaml
}
```

<details>
  <summary>Resultado</summary>

  ```
  ---
  apiVersion: source.toolkit.fluxcd.io/v1beta1
  kind: HelmRepository
  metadata:
    name: grafana
    namespace: flux-system
  spec:
    interval: 5m0s
    url: https://grafana.github.io/helm-charts
  ```
</details>

Crear despliegues:

```bash
{
cat <<EOF > ./clusters/demo/monitoring/loki-stack-helmrelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: loki-stack
  namespace: monitoring
spec:
  chart:
    spec:
      chart: loki-stack
      sourceRef:
        kind: HelmRepository
        name: grafana
        namespace: flux-system
      version: 2.4.1
  install: {}
  interval: 1m0s
  values:
    promtail:
      enabled: false
    grafana:
      enabled: true
      sidecar:
        dashboards:
          enabled: true
        datasources:
          enabled: true
    prometheus:
      enabled: true
      nodeExporter:
        enabled: false
      pushgateway:
        enabled: false
      alertmanager:
        enabled: false
      server:
        global:
          scrape_interval: 10s
EOF

cat <<EOF > ./clusters/demo/gitops-series/echobot-helmrelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: echobot
  namespace: gitops-series
spec:
  interval: 1m0s
  chart:
    spec:
      chart: echobot
      version: 0.3.4
      sourceRef:
        kind: HelmRepository
        name: sngular
        namespace: flux-system
  values:
    image:
      tag: v0.2.1
EOF
}
```

Adicionar flux dashboards:

```bash
{
  FLUX_DASHBOARDS_BASE_URL="https://raw.githubusercontent.com/sngular/gitops-flux-series/monitoring/7.2-monitorizacion/dashboards"
  FLUX_DASHBOARDS_CLUSTER="clusters/demo/monitoring/dashboards"
  curl ${FLUX_DASHBOARDS_BASE_URL}/cluster.json > ${FLUX_DASHBOARDS_CLUSTER}/cluster.json
  curl ${FLUX_DASHBOARDS_BASE_URL}/control-plane.json > ${FLUX_DASHBOARDS_CLUSTER}/control-plane.json
}
```

```bash
cat <<EOF > ./clusters/demo/monitoring/dashboards/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: monitoring
commonLabels:
  grafana_dashboard: "1"
configMapGenerator:
- name: grafana-dashboards
  files:
  - control-plane.json
  - cluster.json
EOF
```

Realice un commit con los cambios al repositorio de código:

```bash
{
  git add .
  git commit -m 'Add resources'
  git push origin main
}
```

Sincronizar la información sin esperara al ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

## Acceso a Grafana

Usuario `admin`

Obtener contraseña:

```bash
kubectl get secret --namespace monitoring loki-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Hacer un port forwarding y acceder a la url `http://localhost:3000`:

```bash
kubectl port-forward --namespace monitoring service/loki-stack-grafana 3000:80
```

## Consulta de logs

### Consultas estáticas

Para realizar consultas sobre los logs entrar en la sección `Explore` de Grafana, seleccionar `Loki` como fuente de datos y utilizar el cuadro de texto para introducir las consultas.

Buscar trazas por nombre de la aplicación o por namespace:

```
{job="flux-system/helm-controller"} |= "echobot"
{job="flux-system/helm-controller"} |= "gitops-series"
```

Buscar trazas de error:

```
{job="flux-system/source-controller"} |~ "error"
{job="flux-system/kustomize-controller"} |~ "error"
{job="flux-system/helm-controller"} |~ "error"
{job="flux-system/notification-controller"} |~ "error"
```

Buscar trazaas de una aplicación:

```
# Trazas de error
{app="echobot"} |~ "error"

# Trazas http de un servicio
{app="my-service"} |~ "http"
{app="my-service"} |~ "status=404"
{app="my-service"} |~ "status=500"
{app="my-service"} |~ "status=40.*"
{app="my-service"} |~ "status=50.*"
{app="gitops-webhook"} |~ "path=/webhook"
```

### En tiempo real

Para ver logs en tiempo real entrar en la sección `Explore` de Grafana, seleccionar `Loki` como fuente de datos, introducir la consulta en el cuadro de texto y pulsar el boton `Live` que aparece en la parte superior derecha de la pantalla.

## (Opcional) Desintalar Flux

Utilice el siguiente comando para desintalar flux del cluster:

```bash
flux uninstall --silent
```

> Compruebe que el repositorio en GitHub no ha sido eliminado.

<details>
  <summary>Resultado</summary>

  ```bash
  ► deleting components in flux-system namespace
  ✔ Deployment/flux-system/helm-controller deleted
  ✔ Deployment/flux-system/kustomize-controller deleted
  ✔ Deployment/flux-system/notification-controller deleted
  ✔ Deployment/flux-system/source-controller deleted
  ✔ Service/flux-system/notification-controller deleted
  ✔ Service/flux-system/source-controller deleted
  ✔ Service/flux-system/webhook-receiver deleted
  ✔ NetworkPolicy/flux-system/allow-egress deleted
  ✔ NetworkPolicy/flux-system/allow-scraping deleted
  ✔ NetworkPolicy/flux-system/allow-webhooks deleted
  ✔ ServiceAccount/flux-system/helm-controller deleted
  ✔ ServiceAccount/flux-system/kustomize-controller deleted
  ✔ ServiceAccount/flux-system/notification-controller deleted
  ✔ ServiceAccount/flux-system/source-controller deleted
  ✔ ClusterRole/crd-controller-flux-system deleted
  ✔ ClusterRoleBinding/cluster-reconciler-flux-system deleted
  ✔ ClusterRoleBinding/crd-controller-flux-system deleted
  ► deleting toolkit.fluxcd.io finalizers in all namespaces
  ✔ GitRepository/flux-system/flux-system finalizers deleted
  ✔ Kustomization/flux-system/flux-system finalizers deleted
  ► deleting toolkit.fluxcd.io custom resource definitions
  ✔ CustomResourceDefinition/alerts.notification.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/buckets.source.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/gitrepositories.source.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/helmcharts.source.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/helmreleases.helm.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/helmrepositories.source.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/kustomizations.kustomize.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/providers.notification.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/receivers.notification.toolkit.fluxcd.io deleted
  ✔ Namespace/flux-system deleted
  ✔ uninstall finished
  ```
</details>
