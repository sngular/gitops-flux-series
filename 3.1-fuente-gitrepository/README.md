# 3.1 Fuente GitRepository

En esta sección veremos cómo Flux es capaz de desplegar en el cluster recursos alojados en un repositorio git.

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >=0.13.2

## Clonar el repositorio con la guía de pasos

```bash
git clone https://github.com/sngular/gitops-flux-series.git
```

## Instalar Flux en el cluster

```bash
flux bootstrap github \
  --owner=sngular \
  --repository=gitops-flux-series-demo \
  --branch=main \
  --private \
  --path=./cluster/namespaces
```

## Clonar repositorio de contenido

```bash
{
  git clone git@github.com:sngular/gitops-flux-series-demo.git
  cd gitops-flux-series-demo
  tree
}
```

## Comprobar el funcionamiento de flux

```bash
kubectl --namespace flux-system get pods
```

## Copiar los manifiestos del namespace gitops-series

```bash
{
    cp -r ../gitops-flux-series/3.1-fuente-gitrepository/manifests/gitops-series cluster/namespaces
    tree
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

Si no desea esperar el tiempo de espera definido en el ciclo de reconciliación puede utilizar los siguienes comandos.

```bash
{
    flux reconcile source git flux-system
    echo
    flux reconcile kustomization flux-system
}
```

## Adicionar la fuente de origen de la aplicación

```bash
{
    mkdir cluster/namespaces/sources/
    cp ../gitops-flux-series/3.1-fuente-gitrepository/manifests/sources/gitrepository-tag.yaml cluster/namespaces/sources/echobot.yaml
    tree
}
```

Agregar los cambios en el repositorio

```bash
{
    git add .
    git commit -m 'Add echobot sources'
    git push origin main
    kubectl get gitrepositories.source.toolkit.fluxcd.io --all-namespaces --watch
}
```

Consultar los cambios detectados por flux

```bash
kubectl get gitrepositories.source.toolkit.fluxcd.io --all-namespaces --watch
```

Puede ser que esté sincronizado el contenido del repositorio, pero todavía el controlador Kustomization no ha realizado su ciclo de reconciliación.

## Configurar semantic version para GitRepository

```bash
{
    cp ../gitops-flux-series/3.1-fuente-gitrepository/manifests/sources/gitrepository-semver.yaml cluster/namespaces/sources/echobot.yaml
    tree
}
```

```bash
{
    git add .
    git commit -m 'Setup semver to echobot sources'
    git push origin main
    kubectl get gitrepositories.source.toolkit.fluxcd.io --all-namespaces --watch
}
```

