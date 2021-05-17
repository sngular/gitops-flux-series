# 3.1 Fuente GitRepository

En esta sección veremos cómo Flux es capaz de desplegar en el cluster recursos alojados en un repositorio git.

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >=0.13.2

## Clonar el repositorio con la guía de pasos

```bash
git clone https://github.com/$GITHUB_USER/gitops-flux-series.git
```

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

  ```
  ► connecting to github.com
  ✔ repository "https://github.com/sngular/gitops-flux-series-demo" created
  ► cloning branch "main" from Git repository "https://github.com/sngular/gitops-flux-series-demo.git"
  ✔ cloned repository
  ► generating component manifests
  ✔ generated component manifests
  ✔ committed sync manifests to "main" ("1bba6481f9261943997b3eac77ddd95f37ad3ffa")
  ► pushing component manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► installing components in "flux-system" namespace
  ✔ installed components
  ✔ reconciled components
  ► determining if source secret "flux-system/flux-system" exists
  ► generating source secret
  ✔ public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDcW5fJvdPje3qMRDoW59hRD/gGIBnPUcEz2fKLJkRAo0tE+q8Suq20Lhmnqb5CB7EvXB1Nl56k62j/K6cMBXW6ERbZy4c47CkeMyee14G8ZdVJbOS3x0pvyl4swp2AFzL6SECPqbrVQZgSQmdbtUaRseS0hC50gOWEypCHY4bo3PQPcXbNhdN/G3oMNn3707+E7A58wsRL2pRsmevjRXIL66108FMT9lPjE0vi7l5JZ32MvuFWwP5rZM39qtLSeXheFa2jpcCBPEczxbdoqijhSzV0PNZqcb8Vbr974WVvaHitAdkhm4/aHJCRhiZuzzpbavoaKNHoNe24oQcIPnuJ
  ✔ configured deploy key "flux-system-main-flux-system-./clusters/demo" for "https://github.com/sngular/gitops-flux-series-demo"
  ► applying source secret "flux-system/flux-system"
  ✔ reconciled source secret
  ► generating sync manifests
  ✔ generated sync manifests
  ✔ committed sync manifests to "main" ("5fff63da327dd6b6b773e02d612f2f663d4c9d49")
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

## Clonar repositorio de contenido

```bash
{
  git clone git@github.com:$GITHUB_USER/gitops-flux-series-demo.git
  cd gitops-flux-series-demo
  tree
}
```

<details>
  <summary>Resultado</summary>

  ```
  Cloning into 'gitops-flux-series-demo'...
  remote: Enumerating objects: 13, done.
  remote: Counting objects: 100% (13/13), done.
  remote: Compressing objects: 100% (6/6), done.
  remote: Total 13 (delta 0), reused 13 (delta 0), pack-reused 0
  Receiving objects: 100% (13/13), 17.43 KiB | 17.43 MiB/s, done.
  .
  └── clusters
      └── demo
          └── flux-system
              ├── gotk-components.yaml
              ├── gotk-sync.yaml
              └── kustomization.yaml

  3 directories, 3 files
  ```
</details>

## Comprobar el funcionamiento de flux

```bash
kubectl --namespace flux-system get pods
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                                       READY   STATUS    RESTARTS   AGE
  notification-controller-5c4d48f476-q7xz2   1/1     Running   0          9m25s
  helm-controller-85bfd4959d-7bxj7           1/1     Running   0          9m26s
  kustomize-controller-6977b8cdd4-p7jnm      1/1     Running   0          9m26s
  source-controller-85fb864746-lsmq4         1/1     Running   0          9m25s
  ```
</details>

## Copiar los manifiestos del namespace gitops-series

Crear carpeta gitops-series:
```bash
mkdir -p ./clusters/demo/gitops-series
```

Crear el fichero del namespace:

```bash
cat <<EOF > ./clusters/demo/gitops-series/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-series
EOF
```

Subir el fichero al repositorio:

```bash
{
    git add .
    git commit -m 'Add gitops series namespace'
    git push origin main
}
```

Comprobar que se ha realizado la sincronización con el repositorio del cluster

```bash
{
    flux get sources git
    echo
    kubectl get namespaces
}
```

<details>
  <summary>Resultado</summary>

  ```
  NAME            READY   MESSAGE                                                            REVISION                                        SUSPENDED
  flux-system     True    Fetched revision: main/dee7b07acf15605ea40f9c3530b0c2d371a791e9    main/dee7b07acf15605ea40f9c3530b0c2d371a791e9   False
  
  NAME              STATUS   AGE
  default           Active   27m
  kube-system       Active   27m
  kube-public       Active   27m
  kube-node-lease   Active   27m
  flux-system       Active   17m
  gitops-series     Active   4s
  ```

</details>

Si no desea esperar el tiempo de espera definido en el ciclo de reconciliación puede utilizar los siguienes comandos:

```bash
flux reconcile kustomization flux-system --with-source
```

<details>
  <summary>Resultado</summary>

  ```
  ► annotating GitRepository flux-system in flux-system namespace
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/dee7b07acf15605ea40f9c3530b0c2d371a791e9
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/dee7b07acf15605ea40f9c3530b0c2d371a791e9
  ```

</details>


## Añadir la fuente de origen de la aplicación

Crear carpeta gitops-series:
```bash
mkdir ./clusters/demo/sources/
```

Crear el fichero del namespace:

```bash
cat <<EOF > clusters/demo/sources/echobot.yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: echobot
  namespace: gitops-series
spec:
  interval: 1m0s
  url: https://github.com/sngular/gitops-echobot.git
  ref:
    tag: v0.1.1
  ignore: |
    # exclude all
    /*
    # include deploy dir
    !/deploy
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: echobot
  namespace: gitops-series
spec:
  interval: 10m0s
  path: ./deploy
  prune: true
  sourceRef:
    kind: GitRepository
    name: echobot
  validation: client
EOF
```

Comprobar el árbol de ficheros:

```bash
tree
```

<details>
  <summary>Resultado</summary>

  ```
  .
  └── clusters
      └── demo
          ├── flux-system
          │   ├── gotk-components.yaml
          │   ├── gotk-sync.yaml
          │   └── kustomization.yaml
          ├── gitops-series
          │   └── namespace.yaml
          └── sources
              └── echobot.yaml
  ```

</details>

Agregar los cambios en el repositorio

```bash
{
    git add .
    git commit -m 'Add echobot sources'
    git push origin main
}
```

Consultar los cambios detectados por flux

```bash
kubectl get gitrepositories.source.toolkit.fluxcd.io --all-namespaces --watch
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE       NAME          URL                                                    READY   STATUS                                                              AGE
  flux-system     flux-system   ssh://git@github.com/sngular/gitops-flux-series-demo   True    Fetched revision: main/e0a9b9944729a9be55fe5f999a6524ca6d171026     59m
  gitops-series   echobot       https://github.com/sngular/gitops-echobot.git          True    Fetched revision: v0.1.1/98af1d5298ba2fb8bfda3b363d1c661a2116de8d   16m
  ```

</details>

Puede ser que esté sincronizado el contenido del repositorio, pero todavía el controlador Kustomization no haya realizado su ciclo de reconciliación.

## Configurar semantic version para GitRepository

```bash
cat <<EOF > ./clusters/demo/sources/echobot.yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: echobot
  namespace: gitops-series
spec:
  interval: 1m0s
  url: https://github.com/sngular/gitops-echobot.git
  ref:
    semver: ">=0.1.0 <1.0.0"
  ignore: |
    # exclude all
    /*
    # include deploy dir
    !/deploy
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: echobot
  namespace: gitops-series
spec:
  interval: 10m0s
  path: ./deploy
  prune: true
  sourceRef:
    kind: GitRepository
    name: echobot
  validation: client
EOF
```

Comprobar los cambios en el manifiesto de despliegue:

```bash
git diff
```

<details>
  <summary>Resultado</summary>

  ```bash
  diff --git a/clusters/demo/sources/echobot.yaml b/clusters/demo/sources/echobot.yaml
  index b36d851..2e779dc 100644
  --- a/clusters/demo/sources/echobot.yaml
  +++ b/clusters/demo/sources/echobot.yaml
  @@ -8,7 +8,7 @@ spec:
    interval: 1m0s
    url: https://github.com/sngular/gitops-echobot.git
    ref:
  -    tag: v0.1.1
  +    semver: ">=0.1.0 <1.0.0"
    ignore: |
      # exclude all
      /*
  ```
</details>

Añadir los cambios y hacer un commit al repositorio:

```bash
{
    git add .
    git commit -m 'Setup semver to echobot sources'
    git push origin main
}
```

Consultar los cambios detectados por flux

```bash
kubectl get gitrepositories.source.toolkit.fluxcd.io --all-namespaces --watch
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE       NAME          URL                                                    READY   STATUS                                                              AGE
  flux-system     flux-system   ssh://git@github.com/sngular/gitops-flux-series-demo   True    Fetched revision: main/e0a9b9944729a9be55fe5f999a6524ca6d171026     60m
  gitops-series   echobot       https://github.com/sngular/gitops-echobot.git          True    Fetched revision: v0.1.1/98af1d5298ba2fb8bfda3b363d1c661a2116de8d   17m
  flux-system     flux-system   ssh://git@github.com/sngular/gitops-flux-series-demo   True    Fetched revision: main/d5b12e7733068359bbeb6225fdc1c771043d7195     60m
  flux-system     flux-system   ssh://git@github.com/sngular/gitops-flux-series-demo   True    Fetched revision: main/d5b12e7733068359bbeb6225fdc1c771043d7195     60m
  gitops-series   echobot       https://github.com/sngular/gitops-echobot.git          True    Fetched revision: v0.1.1/98af1d5298ba2fb8bfda3b363d1c661a2116de8d   17m
  gitops-series   echobot       https://github.com/sngular/gitops-echobot.git          Unknown   reconciliation in progress                                          17m
  gitops-series   echobot       https://github.com/sngular/gitops-echobot.git          True      Fetched revision: v0.1.3/4e2444fd6f8fe033249a56f6ac088a883fea0621   17m
  ```

</details>

## (Opcional) Desintalar Flux

Si quieres desinstalar Flux puedes utilizar este comando:

```bash
flux uninstall
```

> Compruebe que el repositorio en GitHub no ha sido eliminado.

<details>
  <summary>Resultado</summary>

  ```
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
