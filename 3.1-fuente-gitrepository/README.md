# 3.1 Fuente GitRepository

En esta sección se mostrará cómo Flux es capaz de desplegar en el cluster recursos alojados en múltiples repositorio git.

Vídeo de la explicación y la demo completa en este [enlace](https://www.youtube.com/watch?v=QBHJPk874AM&list=PLuQL-CB_D1E7gRzUGlchvvmGDF1rIiWkj&index=3).

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >=0.13.2

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
  ✔ committed sync manifests to "main" ("5fa0702bbd4bdd3a0fe6731cf363cecba9227b0f")
  ► pushing component manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► installing components in "flux-system" namespace
  ✔ installed components
  ✔ reconciled components
  ► determining if source secret "flux-system/flux-system" exists
  ► generating source secret
  ✔ public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDOm1vvGKywX+iJy5Td2S+8F55OPYGJFpoE3sY4qck7wefDyV5KqJehqHz/c1E52HeCMo4ecWyugA+QoUkbqAN9db5hFL1uF51J8Sv7jZ8SpqZ2s50u3gX8IfBxWuWuhlekW1yylvJYDAN5mXYqb9GSDf3QfvhrPKsLCscuHJC0ctGzXFfQDJKHSgQ2PXdybaoVHISYNk/icnxCcbLxgIQEBcgBXtB5E4CM2nBi+Xqa1SqJJTZX3+FuEC3/3LoE1MkDKrH6kLyvIrbWm5u6j934b8ZhxXUiUz+YuJ5pJJuLyvXhowbF20XiHY5EgTyo3+1e3BmoQ/bTWcp0ISlQmmfD
  ✔ configured deploy key "flux-system-main-flux-system-./clusters/demo" for "https://github.com/sngular/gitops-flux-series-demo"
  ► applying source secret "flux-system/flux-system"
  ✔ reconciled source secret
  ► generating sync manifests
  ✔ generated sync manifests
  ✔ committed sync manifests to "main" ("724be4ea6bfadd8e37c1c30463c38b2a9daeb9bf")
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

## Clonar el repositorio de contenido

```bash
{
  git clone git@github.com:$GITHUB_USER/gitops-flux-series-demo.git
  cd gitops-flux-series-demo
}
```

```bash
tree

.
└── clusters
    └── demo
        └── flux-system
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml

3 directories, 3 files
```

## Comprobar el funcionamiento de flux

```bash
kubectl get pods \
  --namespace flux-system
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                                       READY   STATUS    RESTARTS   AGE
  helm-controller-5df867d77f-kh8js           1/1     Running   0          3m51s
  kustomize-controller-66467d9c5d-9cb5l      1/1     Running   0          3m52s
  notification-controller-85f6bf878f-4pl7n   1/1     Running   0          3m51s
  source-controller-f47cf45bf-29pdp          1/1     Running   0          3m51s
  ```
</details>

## Crear los manifiestos del namespace gitops-series

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

```bash
tree
.
└── clusters
    └── demo
        ├── flux-system
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml
        └── gitops-series
            └── namespace.yaml

4 directories, 4 files
```

Adicionar el fichero al repositorio:

```bash
{
  git add .
  git commit -m 'Add gitops series namespace'
  git push origin main
}
```

Si no desea esperar el tiempo de espera definido en el ciclo de reconciliación puede utilizar el siguiente comandos:

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
  ✔ fetched revision main/07b4d29335294f2299b9f3105abc83d34258181f
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/07b4d29335294f2299b9f3105abc83d34258181f
  ```
</details>

Comprobar que se ha realizado la sincronización con el repositorio:

```bash
flux get sources git --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE       NAME            READY   MESSAGE                                                         REVISION                                        SUSPENDED
  flux-system     flux-system     True    Fetched revision: main/07b4d29335294f2299b9f3105abc83d34258181f main/07b4d29335294f2299b9f3105abc83d34258181f   False
  ```
</details>

## Añadir la fuente de origen de la aplicación

Crear carpeta para almacenar las fuentes de información:

```bash
mkdir ./clusters/demo/sources/
```

Crear el fichero con la fuente donde se encuentran los manifiestos de despliegue de la aplicación `echobot`:

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

  5 directories, 5 files
  ```

</details>

Agregar los cambios en el repositorio:

```bash
{
  git add .
  git commit -m 'Add echobot sources'
  git push origin main
}
```

Consultar los cambios detectados por flux:

```bash
watch -n1 "flux get sources git --all-namespaces"
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE       NAME            READY   MESSAGE                                                                 REVISION                                        SUSPENDED
  flux-system     flux-system     True    Fetched revision: main/763f776ba34a74f398828140d0a8b1b765c723d3         main/763f776ba34a74f398828140d0a8b1b765c723d3   False
  gitops-series   echobot         True    Fetched revision: v0.1.1/98af1d5298ba2fb8bfda3b363d1c661a2116de8d       v0.1.1/98af1d5298ba2fb8bfda3b363d1c661a2116de8d False
  ```
</details>

> Si demora en ver los cambios utilice el siguiente comando para acelerar el proceso de sincronización:

```bash
flux reconcile kustomization flux-system --with-source
```

Comprobar que la aplicación se encuentra en ejecución:

```bash
{
  kubectl get pods --namespace gitops-series
  echo
  kubectl get pods \
    --namespace gitops-series \
    --output jsonpath='{.items[0].spec.containers[0].image}'
}
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                       READY   STATUS    RESTARTS   AGE
  echobot-58f7955dd4-htzbk   1/1     Running   0          9m52s

  ghcr.io/sngular/gitops-echobot:v0.1.1
  ```

</details>

## Utilizar semantic version para GitRepository

Utilizar el siguiente fragmento de código para modificar el tipo de referencia de la fuente GitRepository:

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

Añadir los cambios al repositorio:

```bash
{
  git add .
  git commit -m 'Setup semver to echobot sources'
  git push origin main
}
```

Consultar los cambios detectados por flux:

```bash
watch -n1 "flux get sources git --all-namespaces"
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE       NAME            READY   MESSAGE                                                                 REVISION                                        SUSPENDED
  flux-system     flux-system     True    Fetched revision: main/47219f0e841e1906f0c1b6a0c3fd2d40ed8139b9         main/47219f0e841e1906f0c1b6a0c3fd2d40ed8139b9   False
  gitops-series   echobot         True    Fetched revision: v0.2.0/7874f56f439b844d11d17c3be8acc41fefd0af31       v0.2.0/7874f56f439b844d11d17c3be8acc41fefd0af31 False
  ```
</details>

> Si demora en ver los cambios utilice el siguiente comando para acelerar el proceso de sincronización:

```bash
flux reconcile kustomization flux-system --with-source
```

Comprobar los cambios en la aplicación:

```bash
{
  kubectl get pods --namespace gitops-series
  echo
  kubectl get pods \
    --namespace gitops-series \
    --output jsonpath='{.items[0].spec.containers[0].image}'
}
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                       READY   STATUS    RESTARTS   AGE
  echobot-59df77f67f-5fj5s   1/1     Running   0          2m17s
  echobot-59df77f67f-dgkh7   1/1     Running   0          38s
  echobot-59df77f67f-h5n2m   1/1     Running   0          42s

  ghcr.io/sngular/gitops-echobot:v0.1.3
  ```
</details>

## (Opcional) Desintalar Flux

Si desea desinstalar Flux puede utilizar el siguiente comando:

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
