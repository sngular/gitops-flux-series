# 3.1 Fuente GitRepository

En esta sección veremos cómo Flux es capaz de desplegar en un cluster recursos alojados en múltiples repositorio git.

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado Flux >=0.13.2

## Clonar el repositorio con la guía de pasos

``bash
git clone https://github.com/sngular/gitops-flux-series.git
```

## Instalar Flux en el cluster

```bash
flux bootstrap github \
    --owner=mmorejon \
    --repository=gitops-demo \
    --branch=main \
    --personal \
    --private \
    --path=./cluster/namespaces
```

## Clonar repositorio de contenido

```bash
{
    git clone git@github.com:mmorejon/gitops-demo.git
    cd gitops-demo
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
cp ../gitops-flux-series/3.1-fuente-gitrepository/manifests/sources/gitrepository-tag.yaml cluster/namespaces/sources/echobot.yaml
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

Puede ser que esté sincronizado el contenido del repositorio, pero todavía el controlador Kustomization no ha llegado su ciclo de reconciliación

## Configurar semantic version para GitRepository

```bash
cp ../gitops-flux-series/3.1-fuente-gitrepository/manifests/sources/gitrepository-semver.yaml cluster/namespaces/sources/echobot.yaml
```

```bash
{
    git add .
    git commit -m 'Setup semver to echobot sources'
    git push origin main
    kubectl get gitrepositories.source.toolkit.fluxcd.io --all-namespaces --watch
}
```