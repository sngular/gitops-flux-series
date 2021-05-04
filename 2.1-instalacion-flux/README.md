# 2.1 Instalación de Flux

En esta sección serán mostrados los pasos para instalar flux en el cluster de Kubernetes.

## Requisito

* Acceso para administrar un cluster de Kubernetes >=v1.19

## Clonar repositorio de contenido

```bash
git clone https://github.com/sngular/gitops-flux-series.git
```

## Instalación del binario flux

Utilice el siguiente enlace para conocer las versiones disponibles: <https://toolkit.fluxcd.io/get-started/#install-the-flux-cli>

Se recomienda utilizar el siguiente script para la instalación de la última versión de Flux.

```bash
sudo curl -s https://toolkit.fluxcd.io/install.sh | sudo bash

[INFO]  Downloading metadata https://api.github.com/repos/fluxcd/flux2/releases/latest
[INFO]  Using 0.13.2 as release
[INFO]  Downloading hash https://github.com/fluxcd/flux2/releases/download/v0.13.2/flux_0.13.2_checksums.txt
[INFO]  Downloading binary https://github.com/fluxcd/flux2/releases/download/v0.13.2/flux_0.13.2_darwin_amd64.tar.gz
[INFO]  Verifying binary download
[INFO]  Installing flux to /usr/local/bin/flux
```

Comprobar el resultado de la instalación

```bash
flux --version
```

## Estructura del comando

```bash
flux --help | less
```

## Comprobar que el cluster cumple los requisitos

```bash
flux check --pre
```

## Exportar token de GitHub

```bash
export GITHUB_TOKEN=<your-token>
```

## Instalar Flux en el cluster

```bash
flux bootstrap github \
  --owner=sngular \
  --repository=gitops-flux-series-demo \
  --branch=main \
  --private=false \
  --path=./cluster/namespaces
```

Ver los componentes que han sido instalados

```bash
{
  kubectl get namespaces
  echo
  kubectl get pods --namespace flux-system
}
```

Ver los CRD creados

```bash
kubectl get crd | grep fluxcd
```

## Clonar repositorio creado

```bash
{
  git clone git@github.com:sngular/gitops-flux-series-demo.git
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

Copiar manifiestos

```bash
{
  cp -r ../gitops-flux-series/2.1-instalacion-flux/manifests/gitops-series cluster/namespaces
  tree
}
```

Mostrar los logs

```bash
kubectl --namespace flux-system logs --selector app=source-controller --follow
```

Incluir los manifiestos en el repositorio

```bash
{
  git pull origin main
  git add .
  git commit -m 'Add gitops manifests'
  git push origin main
}
```

Observar los pods

```bash
watch -n1 kubectl get pods --namespace gitops-echobot
```

Observar los logs del pod

```bash
kubectl logs -f --namespace gitops-echobot echobot
```

## (Opcional) Desintalar Flux

```bash
flux uninstall
```
