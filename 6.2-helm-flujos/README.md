# 6.2 Tests, actualizaciones y rollbacks con Helm

En esta sección se mostrará la capacidad de Flux para orquestar flujos como el de test, actualización y rollback utilizando [Helm](https://helm.sh/).

Los pasos a realizar serán los siguiente:
  1. Desplegar el servicio echobot (Chart: v0.2.1, servicio: v0.1.3).
  2. Intentar actualizar el chart del echobot a la versión v0.3.4 sin clumplir los requisitos de dependencia con la base de datos (flujo: test -> retries -> rollback).
  3. Deshacer los cambios en el repositorio Git y reflejarlos en el cluster para preparar el siguiente escenario. (flujo: git revert -> sincronización)
  4. Desplegar una instancia de [MongoDB](https://www.mongodb.com/) y actualizar el chart del echobot a v0.3.4 incluyendo una dependencia con la base de datos (flujo:  esperará a que su dependencia esté lista, ejecutará los tests y desplegará correctamente)

Vídeo de la explicación y la demo completa en este [enlace](https://www.youtube.com/watch?v=wQZ01-3vXBI&list=PLuQL-CB_D1E7gRzUGlchvvmGDF1rIiWkj&index=7).

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

## Desplegar el servicio echobot

Añadir al cluster un repositorio de Helm charts como fuente:

Crear carpeta `sources`:
```bash
mkdir -p ./clusters/demo/sources
```

```bash
flux create source helm sngular \
  --url=https://sngular.github.io/gitops-helmrepository/ \
  --interval=5m \
  --namespace=flux-system \
  --export > clusters/demo/sources/sngular-helmrepository.yaml
```

<details>
  <summary>Resultado</summary>

  ```
  ---
  apiVersion: source.toolkit.fluxcd.io/v1beta1
  kind: Helmepository
  metadata:
    name: sngular
    namespace: flux-system
  spec:
    interval: 5m0s
    url: https://sngular.github.io/gitops-helmrepository/
  ```
</details>


Crear la carpeta `gitops-series` para almacenar los manifiestos de despliegue:

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

Crear el fichero `helmrelease` a través del comando `flux create`:

```bash
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
      version: 0.2.1
      sourceRef:
        kind: HelmRepository
        name: sngular
        namespace: flux-system
  values:
    image:
      tag: v0.1.3
EOF
```

Añadir los cambios en el repositorio:

```bash
{
  git add .
  git commit -m 'Add echobot repository and release file'
  git push origin main
}
```

Sincronizar la información sin esperar a al ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

```bash
{
  flux reconcile source helm sngular --namespace=flux-system
  flux reconcile helmrelease echobot --namespace=gitops-series
}
```

<details>
  <summary>Resultado</summary>

  ```
  ► annotating HelmRepository sngular in flux-system namespace
  ✔ HelmRepository annotated
  ◎ waiting for HelmRepository reconciliation
  ✔ HelmRepository reconciliation completed
  ✔ fetched revision 36273ffdf3c89d5d9efdb8c7349202180eea1beb
  ► annotating HelmRelease echobot in gitops-series namespace
  ✔ HelmRelease annotated
  ◎ waiting for HelmRelease reconciliation
  ✔ HelmRelease reconciliation completed
  ✔ applied revision 0.2.1
  ```
</details>

Comprobar que el estado de los recursos desplegados es correcto, el campo `READY` debe ser `True`:

```bash
flux get all --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE       NAME                            READY   MESSAGE                                                         REVISION                                        SUSPENDED
  flux-system     gitrepository/flux-system       True    Fetched revision: main/adfa5f867305835559147e581ba567a3ec669fad main/adfa5f867305835559147e581ba567a3ec669fad   False
  
  NAMESPACE       NAME                    READY   MESSAGE                                                         REVISION                                        SUSPENDED
  flux-system     helmrepository/sngular  True    Fetched revision: 36273ffdf3c89d5d9efdb8c7349202180eea1beb      36273ffdf3c89d5d9efdb8c7349202180eea1beb        False
  
  NAMESPACE       NAME                            READY   MESSAGE                 REVISION        SUSPENDED
  flux-system     helmchart/gitops-series-echobot True    Fetched revision: 0.2.1 0.2.1           False
  
  NAMESPACE       NAME                    READY   MESSAGE                                 REVISION        SUSPENDED
  gitops-series   helmrelease/echobot     True    Release reconciliation succeeded        0.2.1           False
  
  NAMESPACE       NAME                            READY   MESSAGE                                                         REVISION                                        SUSPENDED
  flux-system     kustomization/flux-system       True    Applied revision: main/adfa5f867305835559147e581ba567a3ec669fad main/adfa5f867305835559147e581ba567a3ec669fad   False
  ```
</details>

Listar los pods del servicio desplegado:

```bash
kubectl get pods --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                      READY   STATUS    RESTARTS   AGE
  echobot-bcfb77fcd-cqnqj   1/1     Running   0          7m42s
  ```
</details>


## Actualizar el echobot

La nueva versión del servicio `echobot` (`0.2.1`) es capaz de introducir datos en una base de datos [MongoDB](https://www.mongodb.com/) si la variable de entorno `OUTPUT_TYPE` tiene el valor `mongodb`. Además la nueva versión del chart (`0.3.4`) ejecuta tests para comprobar la conexión con la base de datos. 

Se actualizará el servicio `echobot` a la versión `0.2.1` y su `HelmRelease` a la versión `0.3.4`, además se añadirán los parámetros necesarios para conectar con una base de datos. Lo que debe ocurrir:

- Al aplicar la actualización de la versión `v0.3.4` se desplegará el pod y se ejecutarán los tests.
- Al no estar desplegada la base de datos los tests fallarán.
- Tras 3 reintentos de ejecución de los tests se hará un rollback automático a la configuración anterior, es decir, se volverá a la `HelmRelease` en su versión `0.2.1`.

Se dejarán preparadas las credenciales de la base de datos ya que son un campo requerido para el despliegue y queremos evitar que falle por ello:

```bash
kubectl create secret generic echobot-mongodb-uri \
  --namespace gitops-series \
  --from-literal="uri=mongodb://echobot:gitops@mongodb/logdb"
```

Crear el manifiesto de actualización con los cambios para controlar la actualización:

```bash
cat <<EOF > clusters/demo/gitops-series/echobot-helmrelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: echobot
  namespace: gitops-series
spec:
  chart:
    spec:
      chart: echobot
      sourceRef:
        kind: HelmRepository
        name: sngular
        namespace: flux-system
      version: 0.3.4
  interval: 1m0s
  test:
    enable: true
  upgrade:
    remediation:
      retries: 3
  values:
    image:
      tag: v0.2.1
    env:
    - name: OUTPUT_TYPE
      value: mongodb
    - name: MESSAGE
      value: Hola MongoDB!
    - name: MONGODB_DATABASE
      value: "logdb"
    mongodb:
      existingSecret: echobot-mongodb-uri
EOF
```

Comprobar los cambios respecto al fichero anterior:

```bash
git diff
```

<details>
  <summary>Resultado</summary>

  ```bash
  diff --git a/clusters/demo/gitops-series/echobot-helmrelease.yaml b/clusters/demo/gitops-series/echobot-helmrelease.yaml
  index dfa77ab..51db8e8 100644
  --- a/clusters/demo/gitops-series/echobot-helmrelease.yaml
  +++ b/clusters/demo/gitops-series/echobot-helmrelease.yaml
  @@ -5,15 +5,29 @@ metadata:
     name: echobot
     namespace: gitops-series
   spec:
  -  interval: 1m0s
     chart:
       spec:
         chart: echobot
  -      version: 0.2.1
         sourceRef:
           kind: HelmRepository
           name: sngular
           namespace: flux-system
  +      version: 0.3.4
  +  interval: 1m0s
  +  test:
  +    enable: true
  +  upgrade:
  +    remediation:
  +      retries: 3
     values:
       image:
  -      tag: v0.1.3
  +      tag: v0.2.1
  +    env:
  +    - name: OUTPUT_TYPE
  +      value: mongodb
  +    - name: MESSAGE
  +      value: Hola MongoDB!
  +    - name: MONGODB_DATABASE
  +      value: "logdb"
  +    mongodb:
  +      existingSecret: echobot-mongodb-uri
  ```
</details>

Añadir los cambios en el repositorio:

```bash
{
  git add .
  git commit -m 'Update echobot helmrelease to v0.3.4'
  git push origin main
}
```

Sincronizar la información sin esperar a al ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

Comprobar el estado de la `HelmRelease`:

```bash
flux get helmrelease --namespace gitops-series
```

```
NAME    READY   MESSAGE                         REVISION        SUSPENDED
echobot False   upgrade retries exhausted       0.2.1           False
```

Se ve cómo el intento de actualización se ha detenido tras 3 intentos y el recurso queda en un estado que informa de ello.

Comprobar el estado de los pods:

```bash
{
  kubectl get pods --namespace gitops-series
  kubectl get deployment echobot -o jsonpath="{..image}" --namespace gitops-series
}
```

```
NAME                      READY   STATUS      RESTARTS   AGE
echobot-test-file         0/1     Completed   0          81s
echobot-test-mongodb      0/1     Error       0          79s
echobot-74c667d48-gt2vn   1/1     Running     0          76s

ghcr.io/sngular/gitops-echobot:v0.1.3
```

Se observa que el pod del servicio sigue en estado `Running` y en la versión `v0.1.3` debido al rollback, mientras que el pod `echobot-test-mongodb` ha quedado en error debido a la ausencia de base de datos.

## Revertir los cambios

Por claridad para ver el comportambiento del siguiente escenario es necesario restaurar el estado previo a la actualización del servicio, para ello:

1. Eliminar los pods de tests:

```bash
kubectl delete pod echobot-test-file echobot-test-mongodb
```

2. Revertir los cambios del último commit:

```bash
{
  git revert HEAD
  git push origin main
  git log --pretty=oneline
}
```

<details>
  <summary>Resultado</summary>

  ```
  [main 22f7fb8] Revert "Update echobot helmrelease to v0.3.4"
   1 file changed, 2 insertions(+), 18 deletions(-)

  Enumerating objects: 11, done.
  Counting objects: 100% (11/11), done.
  Delta compression using up to 8 threads
  Compressing objects: 100% (4/4), done.
  Writing objects: 100% (6/6), 581 bytes | 290.00 KiB/s, done.
  Total 6 (delta 1), reused 0 (delta 0), pack-reused 0
  remote: Resolving deltas: 100% (1/1), completed with 1 local object.
  To github.com:sngular/gitops-flux-series-demo.git
     5a765f8..22f7fb8  main -> main

  22f7fb835a215d8baeb9a9d13f24d335314b61a5 (HEAD -> main, origin/main, origin/HEAD) Revert "Update echobot helmrelease to v0.3.4"
  5a765f8c1379b544e272c7cd2abe6fffae560a23 Update echobot helmrelease to v0.3.4
  adfa5f867305835559147e581ba567a3ec669fad Add echobot repository and release file
  cd5475ffe0b8c50d8b87b3d08783e75ede92ca48 Add Flux sync manifests
  efd06eddfdf15449567e90a1dcba100e61104a0f Add Flux v0.14.2 component manifests
  ```
</details>

3. Sincronizar sin esperara al ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

El estado debe de ser el mismo que antes del intento de actualización:

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE       NAME                            READY   MESSAGE                                                        REVISION                                        SUSPENDED
  flux-system     gitrepository/flux-system       True    Fetched revision: main/22f7fb835a215d8baeb9a9d13f24d335314b61a5main/22f7fb835a215d8baeb9a9d13f24d335314b61a5   False

  NAMESPACE       NAME                    READY   MESSAGE                                                         REVISION                                       SUSPENDED
  flux-system     helmrepository/sngular  True    Fetched revision: 36273ffdf3c89d5d9efdb8c7349202180eea1beb      36273ffdf3c89d5d9efdb8c7349202180eea1beb       False

  NAMESPACE       NAME                            READY   MESSAGE                 REVISION        SUSPENDED
  flux-system     helmchart/gitops-series-echobot True    Fetched revision: 0.2.1 0.2.1           False

  NAMESPACE       NAME                    READY   MESSAGE                                 REVISION        SUSPENDED
  gitops-series   helmrelease/echobot     True    Release reconciliation succeeded        0.2.1           False

  NAMESPACE       NAME                            READY   MESSAGE                                                        REVISION                                        SUSPENDED
  flux-system     kustomization/flux-system       True    Applied revision: main/22f7fb835a215d8baeb9a9d13f24d335314b61a5main/22f7fb835a215d8baeb9a9d13f24d335314b61a5   False
  ```
</details>

## Desplegar la base de datos e introducir la dependencia en el echobot

A continuación se añadirá la fuente del [chart de MongoDB](https://github.com/bitnami/charts/tree/master/bitnami/mongodb):

```bash
flux create source helm bitnami \
  --url=https://charts.bitnami.com/bitnami \
  --interval=30m \
  --namespace=flux-system \
  --export > clusters/demo/sources/bitnami-helmrepository.yaml
```

<details>
  <summary>Resultado</summary>

  ```
  ---
  apiVersion: source.toolkit.fluxcd.io/v1beta1
  kind: HelmRepository
  metadata:
    name: bitnami
    namespace: flux-system
  spec:
    interval: 30m0s
    url: https://charts.bitnami.com/bitnami
  ```
</details>

Según la documentación del [chart de MongoDB](https://github.com/bitnami/charts/tree/master/bitnami/mongodb) es necesario crear un secreto con las credenciales de acceso a la base de datos antes del despliegue:

```bash
kubectl create secret generic mongodb-auth \
  --namespace gitops-series \
  --from-literal="mongodb-password=gitops" \
  --from-literal="mongodb-root-password=flux-gitops-root" \
  --from-literal="mongodb-replica-set-key=flux-gitops-root"
```

Añadimos el manifiesto de despliegue de MongoDB:

```bash
cat <<EOF > clusters/demo/gitops-series/mongodb-helmrelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: mongodb
  namespace: gitops-series
spec:
  chart:
    spec:
      chart: mongodb
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
      version: 10.16.4
  interval: 1m0s
  releaseName: mongodb
  values:
    auth:
      username: echobot
      database: logdb
      existingSecret: mongodb-auth
    persistence.size: 1Gi
EOF
```

Actualizar el echobot incluyendo la dependencia con la base de datos:

```bash
cat <<EOF > clusters/demo/gitops-series/echobot-helmrelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: echobot
  namespace: gitops-series
spec:
  chart:
    spec:
      chart: echobot
      sourceRef:
        kind: HelmRepository
        name: sngular
        namespace: flux-system
      version: 0.3.4
  interval: 1m0s
  test:
    enable: true
  upgrade:
    remediation:
      retries: 3
  dependsOn:
    - name: mongodb  # Dependencia
  values:
    env:
    - name: OUTPUT_TYPE
      value: mongodb
    - name: MESSAGE
      value: Hola MongoDB!
    - name: MONGODB_DATABASE
      value: "logdb"
    image:
      tag: v0.2.1
    mongodb:
      existingSecret: echobot-mongodb-uri
EOF
```

```bash
git diff
```

<details>
  <summary>Resultado</summary>

  ```
  diff --git a/clusters/demo/gitops-series/echobot-helmrelease.yaml b/clusters/demo/gitops-series/echobot-helmrelease.yaml
  index 208fa41..8f24c03 100644
  --- a/clusters/demo/gitops-series/echobot-helmrelease.yaml
  +++ b/clusters/demo/gitops-series/echobot-helmrelease.yaml
  @@ -12,6 +12,24 @@ spec:
           kind: HelmRepository
           name: sngular
           namespace: flux-system
  -      version: 0.2.1
  +      version: 0.3.4
     interval: 1m0s
  -
  +  test:
  +    enable: true
  +  upgrade:
  +    remediation:
  +      retries: 3
  +  dependsOn:
  +    - name: mongodb  # Dependencia
  +  values:
  +    env:
  +    - name: OUTPUT_TYPE
  +      value: mongodb
  +    - name: MESSAGE
  +      value: Hola MongoDB!
  +    - name: MONGODB_DATABASE
  +      value: "logdb"
  +    image:
  +      tag: v0.2.1
  +    mongodb:
  +      existingSecret: echobot-mongodb-uri
  ```
</details>

Añadir los cambios en el repositorio:

```bash
{
  git add .
  git commit -m 'Deploy database and include echobot dependency'
  git push origin main
}
```

Sincronizar los cambios sin esperara al ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

Comprobar el estado de la `HelmRelease` del servicio `echobot` mientras se despliega la base de datos:

```bash
flux get hr --namespace gitops-series
```
<details>
  <summary>Resultado</summary>

  ```
  NAME    READY   MESSAGE                                         REVISION        SUSPENDED
  echobot False   dependency 'gitops-series/mongodb' is not ready 0.2.1           False
  mongodb True    Release reconciliation succeeded                10.16.4         False
  ```
</details>

Cuando el estado del despliegue la base de datos es `READY` comienza la actualización del `echobot`. Primero se despliega el pod con la nueva versión y después se realizan los tests, que al completarse correctamente dan por exitosa la operación de actualización:

```bash
flux get helmrelease --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                       READY   STATUS      RESTARTS   AGE
  mongodb-77c47ccd9f-hzgf8   1/1     Running     0          48s
  echobot-7457f689b6-4nztf   1/1     Running     0          17s
  echobot-test-file          0/1     Completed   0          15s
  echobot-test-mongodb       0/1     Completed   0          14s
  ```
</details>

Finalmente la versión de la `HelmRelease` del servicio `echobot` corresponde con la esperada (`0.3.4`):

```bash
flux get helmrelease --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  NAME    READY   MESSAGE                                 REVISION        SUSPENDED
  mongodb True    Release reconciliation succeeded        10.16.4         False
  echobot True    Release reconciliation succeeded        0.3.4           False
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

## Referencias

https://github.com/bitnami/charts/tree/master/bitnami/mongodb
