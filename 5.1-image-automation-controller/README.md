# 5.1 Image Automation Controller

En esta sección exploramos los pasos necesarios para configurar Flux y aplicar entrega continua (Continuous Delivery) a nuestros servicios de Kubernetes.

Vídeo de la explicación y la demo completa en este [vídeo](https://youtube.com/sngular/playlist).

Y si te interesa puedes ver más contenido de esta serie en nuestra [playlist de Youtube](https://youtube.com/sngular/playlist).

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >=0.13.2 - [instrucciones](../2.1-instalacion-flux/readme#instalación-del-binario-flux)

## Exportar token de GitHub

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

## Instalar Flux en el cluster

Al comando de instalación habitual se ha añadido el flag `--components-extra` para incluir los controladores que nos ayudarán a automatizar el despliegue de imágenes.

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=gitops-flux-series-demo \
  --branch=main \
  --private=false \
  --path=./cluster/namespaces \
  --components-extra=image-reflector-controller,image-automation-controller
```
<details>
  <summary>Resultado</summary>

  ```bash
  ► connecting to github.com
  ► cloning branch "main" from Git repository "https://github.com/sngular/gitops-flux-series-demo.git"
  ✔ cloned repository
  ► generating component manifests
  ✔ generated component manifests
  ✔ component manifests are up to date
  ► installing toolkit.fluxcd.io CRDs
  ◎ waiting for CRDs to be reconciled
  ✔ CRDs reconciled successfully
  ► installing components in "flux-system" namespace
  ✔ installed components
  ✔ reconciled components
  ► determining if source secret "flux-system/flux-system" exists
  ► generating source secret
  ✔ public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDSTrIKbYAWLUjcG7ec6lWJ2KACfF5YB5KqpQcN+LmxkSYmJbFPBmlzZdtIUEvZcAORJYeMKvk+iAcZC6rPn0OCBKp3ypOiMC5HnF5Lnn4XPt1+Nwx30mC72RzkheFm+K3Q0kTySAi8QdKy94aWqBVpTdZzkJ0woNHJg/aL3gQnofXueiczwkMvB2B6x4vgdbBgLOrRl7YhtGz0B6e9a7U4EEBoPdzjti/w7OAQnOpCZ80TwYcuFCioPE0q2i3BgKLvt0x9rBikzuOSgqKFfoAy3zPETgWZ0kPSbHby3lv+NfwWaLVULVpkpNQTwxBbMJVDcwKuyTUacSGeZcUzS2mB
  ✔ configured deploy key "flux-system-main-flux-system-./cluster/namespaces" for "https://github.com/sngular/gitops-flux-series-demo"
  ► applying source secret "flux-system/flux-system"
  ✔ reconciled source secret
  ► generating sync manifests
  ✔ generated sync manifests
  ✔ sync manifests are up to date
  ► applying sync manifests
  ✔ reconciled sync configuration
  ◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
  ✔ Kustomization reconciled successfully
  ► confirming components are healthy
  ✔ image-reflector-controller: deployment ready
  ✔ image-automation-controller: deployment ready
  ✔ source-controller: deployment ready
  ✔ kustomize-controller: deployment ready
  ✔ helm-controller: deployment ready
  ✔ notification-controller: deployment ready
  ✔ all components are healthy
  ```
</details>

Comprobar que el despliegue se ha realizado correctamente.

```bash
kubectl get pods --namespace flux-system
```


<details>
  <summary>Resultado</summary>

  ```
  NAME                                           READY   STATUS    RESTARTS   AGE
  helm-controller-85bfd4959d-bfvnf               1/1     Running   0          3m13s
  image-reflector-controller-55fb577bf9-kngjc    1/1     Running   0          3m13s
  image-automation-controller-776465b9b6-q5rz4   1/1     Running   0          3m13s
  kustomize-controller-6977b8cdd4-xncbj          1/1     Running   0          3m13s
  source-controller-85fb864746-xhrvb             1/1     Running   0          3m12s
  notification-controller-5c4d48f476-28g7q       1/1     Running   0          3m13s
  ```
</details>

## Clonar repositorio creado

Clonar el repositorio que Flux está sincronizando con el cluster. 

```bash
{
  git clone git@github.com:$GITHUB_USER/gitops-flux-series-demo.git
  cd gitops-flux-series-demo
}
```

## Desplegar el servicio `echobot`

Vamos a desplegar el servicio `echobot` en una versión menos actual de la que existe en el repositorio de imágenes para después automatizar su actualización.

Crear carpeta gitops-series:
```bash
mkdir -p ./cluster/namespaces/gitops-series
```

Crear el manifiesto del namespace `gitops-series`:

```bash
cat <<EOF > ./cluster/namespaces/gitops-series/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-series
EOF
```

Crear el manifiesto del servicio de prueba:

```bash
cat <<EOF > ./cluster/namespaces/gitops-series/echobot-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echobot
  namespace: gitops-series
  labels:
    name: echobot
spec:
  replicas: 1
  selector:
    matchLabels:
      name: echobot
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: echobot
    spec:
      containers:
        - name: echobot
          image: ghcr.io/sngular/gitops-echobot:v0.1.0
          env:
            - name: CHARACTER
              value: "Esperando la actualización de imagen automágica!"
            - name: SLEEP
              value: "3s"
          resources:
            requests:
              cpu: 10m
              memory: 30Mi
            limits:
              cpu: 10m
              memory: 30Mi
EOF
```

Añadir manifiestos al repositorio:

```bash
{
  git add .
  git commit -m 'Add gitops manifests'
  git push origin main
}
```

Esperar a que el pod se encuentre en estado `Running`.

```bash
watch -n1 kubectl get pods --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                       READY   STATUS              RESTARTS   AGE
  echobot-6786b99558-4rvm6   0/1     ContainerCreating   0          11s
  ```
</details>


Comprobar la versión de la imagen desplegada:

```bash
kubectl get deployment echobot -o jsonpath="{..image}" --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  ghcr.io/sngular/gitops-echobot:v0.1.0
  ```

</details>

## Configurar el repositorio de la imagen

El primer paso para configurar la actualización automática de la imagen es indicarle a Flux en qué repositorio se encuentra la imagen del servicio `echobot`. Una vez hecho esto Flux escaneará el repositorio en busca de nuevas etiquetas.

Para indicarle a Flux cual es el repositorio es necesario crear el recurso `ImageRepository`:

```bash
cat <<EOF > ./cluster/namespaces/gitops-series/imagerepository.yaml
apiVersion: image.toolkit.fluxcd.io/v1alpha2
kind: ImageRepository
metadata:
  name: echobot-repo
  namespace: gitops-series
spec:
  interval: 1m0s
  image: ghcr.io/sngular/gitops-echobot
EOF
```

```bash
{
  git add .
  git commit -m 'Add imagerepository'
  git push origin main
}
```

Esperar a que se realice la sincronización con del repositorio o indicarle a Flux que realice el ciclo de reconciliación de manera inmediata:

```bash
flux reconcile source git flux-system
```

<details>
  <summary>Resultado</summary>

  ```
  ► annotating GitRepository flux-system in flux-system namespace
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/d2bb77a311afd7412737251e98eb78693826aa51
  ```
</details>

Comprobar que se ha creado el `ImageRepository`, que Flux lo ha escaneado y ha encontrado tags:

```bash
# utilizando flux cli (recomendado)
flux get image repository echobot-repo --namespace gitops-series

# o la api de kubernetes
kubectl get imagerepositories echobot-repo
```

<details>
  <summary>Resultado</summary>

  ```
  NAME            READY   MESSAGE                         LAST SCAN                 SUSPENDED
  echobot-repo    True    successful scan, found 4 tags   2021-05-13T20:28:08+02:00 False
  ```
</details>

## Configurar la política de actualización

Ahora se va a desplegar el recurso `ImagePolicy` que nos ayudará a filtrar las imagenes encontradas en el `ImageRepository` y a aplicar un criterio de selección. Por ejemplo, buscaremos la etiqueta de la imagen más reciente cuya versión semántica sea mayor o igual a 0.1.0 (`>=0.1.0`).

Crear el fichero que contiene el recurso  `ImagePolicy` y subir los cambios a nuestro repositorio:

```bash
cat <<EOF > ./cluster/namespaces/gitops-series/imagepolicy.yaml
apiVersion: image.toolkit.fluxcd.io/v1alpha2
kind: ImagePolicy
metadata:
  name: echobot-policy
  namespace: gitops-series
spec:
  imageRepositoryRef:
    name: echobot-repo
  policy:
    semver:
      range: '>=0.1.0 <1.0.0'
EOF
```

```bash
{
  git add .
  git commit -m 'Add imagepolicy'
  git push origin main
}
```

Esperar a que se realice la sincronización con del repositorio o indicarle a Flux que realice el ciclo de reconciliación de manera inmediata:

```bash
flux reconcile source git flux-system
```

<details>
  <summary>Resultado</summary>

  ```
  ► annotating GitRepository flux-system in flux-system namespace
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/2c996c54d54779d886fef7ee7332600c181b43a1
  ```
</details>


Comprobar las imágenes detectadas que cumplen la política que hemos especificado (`>=0.1.0 <1.0.0`):

```bash
# utilizando flux cli (recomendado)
flux get image policy echobot-policy --namespace gitops-series

# o la api de kubernetes
kubectl get imagepolicy echobot-policy --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  NAME            READY   MESSAGE                                                                    LATEST IMAGE
  echobot-policy  True    Latest image tag for 'ghcr.io/sngular/gitops-echobot' resolved to: v0.1.3  ghcr.io/sngular/gitops-echobot:v0.1.3
  ```
</details>

## Configurar el despliegue de la nueva etiqueta

Para que Flux pueda actualizar la imagen del servicio `echobot` es necesario:

1. Crear un recurso `GitRepository` con el que le indicaremos a Flux dónde se encuentra el manifiesto de despliegue del `echobot`.
2. Añadir un marcador en el manifiesto de despliegue del servicio `echobot` para indicarle a Flux qué política debe aplicar al actualizar la imagen.
3. Crear un recurso de tipo `ImageUpdateAutomation`. Flux lo utilizará para actualizar mediante un commit el manifiesto de despliegue (contenido en el `GitRepository`) con la nueva etiqueta de la imagen. Además podremos personalizar el mensaje del commit que hará Flux con información útil. 


##### 1. Crear `GitRepository`

---
**TODO**: Es necesario crear credenciales para que flux pueda realizar el commit al repositorio en el paso del ImageUpdateAutomation.

> Reutilizar el secreto de flux-system???

```bash
kubectl get secret --namespace flux-system flux-system -o yaml | sed -e 's/name: flux-system/name: gitops-repo-creds/' -e 's/namespace: flux-system/namespace: gitops-series/' > cluster/namespaces/gitops-series/gitrepository-creds.yaml
```
---


```bash
cat <<EOF > ./cluster/namespaces/gitops-series/gitrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: gitops-repo
  namespace: gitops-series
spec:
  interval: 1m
  url: https://github.com/sngular/gitops-flux-series-demo.git
  ref:
    branch: main
  ignore: |
    # exclude all
    /*
    # include dir
    !/cluster/namespaces/
EOF
```

```bash
{
  git add .
  git commit -m 'Add gitrepository'
  git push origin main
}
```

Esperar a que se realice la sincronización con del repositorio o indicarle a Flux que realice el ciclo de reconciliación de manera inmediata:

```bash
flux reconcile source git flux-system
```

<details>
  <summary>Resultado</summary>

  ```
  ► annotating GitRepository flux-system in flux-system namespace
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/3aaeb49904ca242275ca4d9839d8bd214bcfdedf
  ```
</details>

Comprobar que Flux ha descargado el contenido del repositorio:

```bash
# utilizando flux cli (recomendado)
flux get sources git --namespace gitops-series

# o la api de kubernetes
kubectl get gitrepositories --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  NAME            READY   MESSAGE                                                            REVISION                                        SUSPENDED
  gitops-repo     True    Fetched revision: main/3aaeb49904ca242275ca4d9839d8bd214bcfdedf    main/3aaeb49904ca242275ca4d9839d8bd214bcfdedf   False
  ```
</details>

##### 2. Añadir marcador para aplicar una política

El formato del marcador sigue las siguientes reglas:

- Si la imagen y el tag de la misma están definidas en la misma línea se utiliza:

```yaml
# {"$imagepolicy": "<policy-namespace>:<policy-name>"}

image: ghcr.io/sngular/gitops-echobot:v0.1.0  # {"$imagepolicy": "gitops-series:echobot:tag"}
```

- Si la imagen y la etiqueta están declaradas en líneas diferentes poner en cada una un marcador indicándolo:

```yaml
# {"$imagepolicy": "<policy-namespace>:<policy-name>:tag"}
# {"$imagepolicy": "<policy-namespace>:<policy-name>:name"}

image:
  repository: ghcr.io/sngular/gitops-echobot  # {"$imagepolicy": "gitops-series:echobot:name"}
  tag: v0.1.0  # {"$imagepolicy": "gitops-series:echobot:tag"}
```

Modifica el manifiesto de despliegue del `echobot` y súbelo al repositorio:

```bash
cat <<EOF > ./cluster/namespaces/gitops-series/echobot-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echobot
  namespace: gitops-series
  labels:
    name: echobot
spec:
  replicas: 1
  selector:
    matchLabels:
      name: echobot
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: echobot
    spec:
      containers:
        - name: echobot
          image: ghcr.io/sngular/gitops-echobot:v0.1.0  # {"$imagepolicy": "gitops-series:echobot-policy"}
          env:
            - name: CHARACTER
              value: "Esperando la actualización de imagen automágica!"
            - name: SLEEP
              value: "3s"
          resources:
            requests:
              cpu: 10m
              memory: 30Mi
            limits:
              cpu: 10m
              memory: 30Mi
EOF
```

Comprobar el camio introducido:

```bash
git diff
```

<details>
  <summary>Resultado</summary>

  ```
  diff --git a/cluster/namespaces/gitops-series/echobot-deployment.yaml b/cluster/namespaces/gitops-series/echobot-deployment.yaml
  index 92dd1d6..1102379 100644
  --- a/cluster/namespaces/gitops-series/echobot-deployment.yaml
  +++ b/cluster/namespaces/gitops-series/echobot-deployment.yaml
  @@ -19,7 +19,7 @@ spec:
       spec:
         containers:
           - name: echobot
  -          image: ghcr.io/sngular/gitops-echobot:v0.1.0
  +          image: ghcr.io/sngular/gitops-echobot:v0.1.0  # {"": "gitops-series:echobot-policy"}
             env:
               - name: CHARACTER
                 value: "Esperando la actualización de imagen automágica!"
  ```
</details>

Subir los cambios al repositorio:

```bash
{
  git add .
  git commit -m 'Add marker to echobot deployment'
  git push origin main
}
```

##### 3. Crear `ImageUpdateAutomation`

```bash
cat <<EOF > ./cluster/namespaces/gitops-series/imageupdateautomation.yaml
apiVersion: image.toolkit.fluxcd.io/v1alpha2
kind: ImageUpdateAutomation
metadata:
  name: echobot-autoupdate
  namespace: gitops-series
spec:
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  interval: 1m
  update:
    strategy: Setters
    path: "./cluster/namespaces"
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: Fluxbot
        email: fluxbot@gitops.com
      messageTemplate: |
        {{ range .Updated.Images -}}
        [dev] Automated image update **{{ \$.AutomationObject }}** to **{{ .Identifier }}**
        {{ end -}}

        Automation name: {{ .AutomationObject }}

        Files:
        {{ range \$filename, \$_ := .Updated.Files -}}
        - {{ \$filename }}
        {{ end -}}

        Objects:
        {{ range \$resource, \$_ := .Updated.Objects -}}
        - {{ \$resource.Kind }} {{ \$resource.Name }}
        {{ end -}}

        Images:
        {{ range .Updated.Images -}}
        - {{.}}
        {{ end -}}
EOF
```

Nota: en el campo `messageTemplate` aparecen escapados el símbolo `$` para que funcione si se copia directamente en shell. En caso de copiar en un fichero reemplazar `\$` por `$`.

Subir los cambios al repositorio:

```bash
{
  git add .
  git commit -m 'Add imageupdateautomation'
  git push origin main
}
```

Esperar a que se realice la sincronización con del repositorio o indicarle a Flux que realice el ciclo de reconciliación de manera inmediata:

```bash
flux reconcile source git flux-system
```

<details>
  <summary>Resultado</summary>

  ```
  ► annotating GitRepository flux-system in flux-system namespace
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/4b6a1cbf35859f62f04294fe4918484826266be9
  ```
</details>

Comprobar las imágenes detectadas que cumplen la política que hemos especificado:

```bash
# utilizando flux cli (recomendado)
flux get image update echobot-autoupdate --namespace gitops-series

# o la api de kubernetes
kubectl get imageupdateautomation echobot-autoupdate
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                    READY   MESSAGE         LAST RUN                        SUSPENDED
  echobot-autoupdate      True    no updates made 2021-05-13T21:03:18+02:00       False
  ```
</details>


Comprobar que en el repositorio indicado en el recurso `GitRepository` habrá un nuevo commit de Flux con los datos de la actualización de la imagen.

Por último, comprobar que se ha actualizado la imagen a la versión más reciente del repositorio de imágenes y que esta coincide con la indicada en la política:

```bash
{
  flux get image policy echobot-policy --namespace gitops-series
  echo
  kubectl get deployment echobot -o jsonpath="{..image}" --namespace gitops-series
}
```

<details>
  <summary>Resultado</summary>

  ```
  NAME            READY   MESSAGE                                                                    LATEST IMAGE
  echobot-policy  True    Latest image tag for 'ghcr.io/sngular/gitops-echobot' resolved to: v0.1.3  ghcr.io/sngular/gitops-echobot:v0.1.3

  ghcr.io/sngular/gitops-echobot:v0.1.3
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
