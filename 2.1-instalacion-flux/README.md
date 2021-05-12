# 2.1 Instalación de Flux

En esta sección serán mostrados los pasos para instalar flux en el cluster de Kubernetes.

## Requisito

* Acceso para administrar un cluster de Kubernetes >=v1.19

## Instalación del binario flux

Utilice el siguiente enlace para conocer las versiones disponibles: <https://toolkit.fluxcd.io/get-started/#install-the-flux-cli>

Se recomienda utilizar el siguiente script para la instalación de la última versión de Flux.

```bash
sudo curl -sL https://toolkit.fluxcd.io/install.sh | sudo bash
```

<details>
  <summary>Resultado</summary>

  ```bash
  [INFO]  Downloading metadata https://api.github.com/repos/fluxcd/flux2/releases/latest
  [INFO]  Using 0.13.4 as release
  [INFO]  Downloading hash https://github.com/fluxcd/flux2/releases/download/v0.13.4/flux_0.13.4_checksums.txt
  [INFO]  Downloading binary https://github.com/fluxcd/flux2/releases/download/v0.13.4/flux_0.13.4_darwin_amd64.tar.gz
  [INFO]  Verifying binary download
  [INFO]  Installing flux to /usr/local/bin/flux
  ```
</details>

Comprobar el resultado de la instalación:

```bash
flux --version
```

<details>
  <summary>Resultado</summary>

  ```bash
  flux version 0.13.4
  ```
</details>

## Estructura del comando flux

Identifique los grupos de comandos que existen en el binario flux:

```bash
flux --help | less
```

<details>
  <summary>Resultado</summary>

  ```bash
  Command line utility for assembling Kubernetes CD pipelines the GitOps way.

Usage:
  flux [command]

  Examples:
    # Check prerequisites
    flux check --pre

    # Install the latest version of Flux
    flux install --version=master

    # Create a source for a public Git repository
    flux create source git webapp-latest \
      --url=https://github.com/stefanprodan/podinfo \
      --branch=master \
      --interval=3m
  ...
  ...
  ...
  ```
</details>

## Comprobar que el cluster cumple los requisitos

Compruebe que cumple con las condiciones para instalar flux:

```bash
flux check --pre
```

<details>
  <summary>Resultado</summary>

  ```bash
  ► checking prerequisites
  ✔ kubectl 1.21.0 >=1.18.0-0
  ✔ Kubernetes 1.19.8-gke.1600 >=1.16.0-0
  ✔ prerequisites checks passed
  ```
</details>

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
  --path=./cluster/namespaces
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
  ✔ committed sync manifests to "main" ("f07664100bb00b85c481d4c703f06879292c5a19")
  ► pushing component manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► installing components in "flux-system" namespace
  ✔ installed components
  ✔ reconciled components
  ► determining if source secret "flux-system/flux-system" exists
  ► generating source secret
  ✔ public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDM10X/KGqYSWFrviPF6ZMRBtT+PV8ypKd8wUPoAccZdPnWlh8G+oc2gwH0jKpYiyKRFuE34RRohW1hgLRjrSiRq1Sd/TpLYSnav61b21Eyz7hBnfIdVn5yI7SKUa+5qDrkGZvn+I8Lwwwm3SagloMIS3dzgH8OsWDNaausSBJupYvwCNA4HbNgm1/wsCfS4EiBagxWmqJZYKQ2L91VInSEiMlcTPILufqjsitJmnLjt4aZ4nIxuHGjeg/8lOxO6dhjj03Cko6JKNXqVLz5gwidhthjJ2LTG2dSTIaxLNfwNWsepH8pI28RxwVrwIYQ1umGkKJcv7u8Uz938gdnaCOV
  ✔ configured deploy key "flux-system-main-flux-system-./cluster/namespaces" for "https://github.com/sngular/gitops-flux-series-demo"
  ► applying source secret "flux-system/flux-system"
  ✔ reconciled source secret
  ► generating sync manifests
  ✔ generated sync manifests
  ✔ committed sync manifests to "main" ("a0c0e07c76e3533f25525685ddd128fde3b4b461")
  ► pushing sync manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► applying sync manifests
  ✔ reconciled sync configuration
  ◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
  ✔ Kustomization reconciled successfully
  ► confirming components are healthy
  ✔ helm-controller: deployment ready
  ✔ notification-controller: deployment ready
  ✔ source-controller: deployment ready
  ✔ kustomize-controller: deployment ready
  ✔ all components are healthy
  ```
</details>

Ver los componentes que han sido instalados:

```bash
{
  kubectl get namespaces
  echo
  kubectl get pods --namespace flux-system
}
```

<details>
  <summary>Resultado</summary>

  ```bash
  NAME                STATUS   AGE
  default             Active   10m
  flux-system         Active   2m46s
  gatekeeper-system   Active   9m46s
  kube-node-lease     Active   10m
  kube-public         Active   10m
  kube-system         Active   10m

  NAME                                       READY   STATUS    RESTARTS   AGE
  helm-controller-5df867d77f-z8j7x           1/1     Running   0          2m35s
  kustomize-controller-576bc889b5-kj8ds      1/1     Running   0          2m32s
  notification-controller-67c46b8cdc-cz9xm   1/1     Running   0          2m31s
  source-controller-94888bb6c-t67zt          1/1     Running   0          2m30s
  ```
</details>

Ver los CRD creados

```bash
kubectl get crd | grep fluxcd
```

<details>
  <summary>Resultado</summary>

  ```bash
  alerts.notification.toolkit.fluxcd.io                             2021-05-12T22:54:47Z
  buckets.source.toolkit.fluxcd.io                                  2021-05-12T22:54:47Z
  gitrepositories.source.toolkit.fluxcd.io                          2021-05-12T22:54:47Z
  helmcharts.source.toolkit.fluxcd.io                               2021-05-12T22:54:47Z
  helmreleases.helm.toolkit.fluxcd.io                               2021-05-12T22:54:49Z
  helmrepositories.source.toolkit.fluxcd.io                         2021-05-12T22:54:50Z
  kustomizations.kustomize.toolkit.fluxcd.io                        2021-05-12T22:54:52Z
  providers.notification.toolkit.fluxcd.io                          2021-05-12T22:54:52Z
  receivers.notification.toolkit.fluxcd.io                          2021-05-12T22:54:52Z
  ```
</details>

## Clonar repositorio creado

```bash
{
  git clone git@github.com:$GITHUB_USER/gitops-flux-series-demo.git
  cd gitops-flux-series-demo
}
```

Consultar la estructura creada

```bash
tree
.
└── cluster
    └── namespaces
        └── flux-system
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml

3 directories, 3 files
```

## Desplegar el primer pod

Crear carpeta gitops-series:
```bash
mkdir -p ./cluster/namespaces/gitops-series
```

Crear el fichero del namespace:

```bash
cat <<EOF > ./cluster/namespaces/gitops-series/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-series
EOF
```

Crear el fichero del pod:

```bash
cat <<EOF > ./cluster/namespaces/gitops-series/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: echobot
  namespace: gitops-series
  labels:
    app: echobot
spec:
  containers:
    - name: message
      image: ghcr.io/sngular/gitops-echobot:v0.1.0
      env:
        - name: CHARACTER
          value: "sngular utiliza gitops en sus entornos"
      resources:
        requests:
          cpu: 10m
          memory: 30Mi
        limits:
          cpu: 10m
          memory: 30Mi
EOF
```

Compruebe la nueva estructura del repositorio:

```bash
tree
.
└── cluster
    └── namespaces
        ├── flux-system
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml
        └── gitops-series
            ├── namespace.yaml
            └── pod.yaml

4 directories, 5 files
```

Mostrar los logs del pod desplegado:

```bash
kubectl logs \
  --namespace flux-system \
  --selector app=source-controller \
  --follow
```

<details>
  <summary>Resultado</summary>

  ```bash
  {"level":"info","ts":"2021-05-12T22:59:00.109Z","logger":"controller.gitrepository","msg":"Reconciliation finished in 1.166725904s, next run in 1m0s","reconciler group":"source.toolkit.fluxcd.io","reconciler kind":"GitRepository","name":"flux-system","namespace":"flux-system"}
  {"level":"info","ts":"2021-05-12T23:00:01.392Z","logger":"controller.gitrepository","msg":"Reconciliation finished in 1.281116458s, next run in 1m0s","reconciler group":"source.toolkit.fluxcd.io","reconciler kind":"GitRepository","name":"flux-system","namespace":"flux-system"}
  ```
</details>

Incluya los ficheros creados en el control de versiones:

```bash
{
  git add .
  git commit -m 'Add gitops series namespace and pod'
  git push origin main
}
```

Acelerar el ciclo de reconciliación:

```bash
{
  flux reconcile source git flux-system
  echo
  flux reconcile kustomization flux-system
}
```

<details>
  <summary>Resultado</summary>

  ```bash
  ► annotating GitRepository flux-system in flux-system namespace
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/33c59431db8b4465cb045743b5725d59150ef9ef
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/33c59431db8b4465cb045743b5725d59150ef9ef
  ```
</details>

Observar los pods

```bash
watch -n1 kubectl get pods --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```bash
  NAME      READY   STATUS    RESTARTS   AGE
  echobot   1/1     Running   0          5m16s
  ```
</details>

Observar los logs del pod

```bash
kubectl logs \
  --namespace gitops-series \
  --selector app=echobot \
  --follow
```

<details>
  <summary>Resultado</summary>

  ```bash
  hostname: echobot - sngular utiliza gitops en sus entornos
  hostname: echobot - sngular utiliza gitops en sus entornos
  hostname: echobot - sngular utiliza gitops en sus entornos
  hostname: echobot - sngular utiliza gitops en sus entornos
  ```
</details>

## (Opcional) Desintalar Flux

Utilice el siguiente comando para desintalar flux del cluster:

```bash
flux uninstall
```

> Compruebe que el repositorio en GitHub no ha sido eliminado.

<details>
  <summary>Resultado</summary>

  ```bash
  Are you sure you want to delete Flux and its custom resource definitions: y█
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