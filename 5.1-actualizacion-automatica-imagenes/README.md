# 5.1 Actualización automática de imágenes

En esta sección serán mostrados los pasos necesarios para configurar Flux y aplicar entrega continua (Continuous Delivery) a nuestros servicios de Kubernetes.

Vídeo de la explicación y la demo completa en este [enlace](https://www.youtube.com/watch?v=wQZ01-3vXBI&list=PLuQL-CB_D1E7gRzUGlchvvmGDF1rIiWkj&index=5).

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >=0.13.2 - [instrucciones](../2.1-instalacion-flux/readme#instalación-del-binario-flux)

## Exportar token de GitHub

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

## Instalar Flux en el cluster

Al comando de instalación habitual se ha añadido el parámetro `--components-extra` para incluir los controladores que permitirán automatizar el despliegue de las imágenes.

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=gitops-flux-series-demo \
  --branch=main \
  --private=false \
  --read-write-key \
  --path=./clusters/demo \
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
  ✔ configured deploy key "flux-system-main-flux-system-./cluster" for "https://github.com/sngular/gitops-flux-series-demo"
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
watch kubectl get pods --namespace flux-system
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

## Desplegar el servicio echobot

Será desplegado el servicio `echobot` en una versión menos actual a la que existe en el repositorio de imágenes para después automatizar su actualización.

Crear carpeta gitops-series:

```bash
mkdir -p ./clusters/demo/gitops-series
```

Crear el manifiesto del namespace `gitops-series`:

```bash
cat <<EOF > ./clusters/demo/gitops-series/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-series
EOF
```

Crear el manifiesto del servicio de echobot:

```bash
cat <<EOF > ./clusters/demo/gitops-series/echobot-deployment.yaml
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

Añadir los manifiestos al repositorio:

```bash
{
  git add .
  git commit -m 'Add gitops series manifests'
  git push origin main
}
```

Espere a que se realice la sincronización con del repositorio o indíquele a Flux que realice el ciclo de reconciliación de manera inmediata:

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
  ✔ fetched revision main/1bcb9c970c9296c7c2525c2f68f19307c0f4a84e
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/1bcb9c970c9296c7c2525c2f68f19307c0f4a84e
  ```
</details>

Esperar a que el pod se encuentre en estado `Running`.

```bash
watch -n1 kubectl get pods --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                       READY   STATUS    RESTARTS   AGE
  echobot-6786b99558-p6dfv   1/1     Running   0          85s
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

El primer paso para configurar la actualización automática de la imagen es indicarle a Flux dónde está almacenada la imagen del servicio `echobot`. Una vez hecho esto Flux escaneará el registro de la imagen en busca de nuevas etiquetas.

Para indicarle a Flux cuál es el registro de contenedores será necesario crear el objeto `ImageRepository`:

```bash
mkdir -p clusters/demo/automation
```

```bash
flux create image repository echobot \
  --image=ghcr.io/sngular/gitops-echobot \
  --interval=1m \
  --namespace=flux-system \
  --export > clusters/demo/automation/echobot-registry.yaml
```

<details>
  <summary>Resultado</summary>

  ```bash
  ---
  apiVersion: image.toolkit.fluxcd.io/v1alpha2
  kind: ImageRepository
  metadata:
    name: echobot
    namespace: flux-system
  spec:
    image: ghcr.io/sngular/gitops-echobot
    interval: 1m0s
  ```

</details>

Adicione el objeto credo al repositorio de código:

```bash
{
  git add .
  git commit -m 'Add echobot image registry'
  git push origin main
}
```

Esperar a que se realice la sincronización con del repositorio o indicarle a Flux que realice el ciclo de reconciliación de manera inmediata:

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
  ✔ fetched revision main/83edfa39b7092027f51c01f6ac6b0fad2b3409d4
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/83edfa39b7092027f51c01f6ac6b0fad2b3409d4
  ```
</details>

Comprobar que se ha creado el objeto `ImageRepository`, que Flux lo ha escaneado y que ha encontrado tags:

```bash
flux get image repository --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE  	NAME   	READY	MESSAGE                      	LAST SCAN                	SUSPENDED
  flux-system	echobot	True 	successful scan, found 4 tags	2021-05-15T19:18:46+02:00	False
  ```
</details>

## Configurar la política de actualización

Ahora se va a desplegar el recurso `ImagePolicy` que permitirá filtrar y ordenar las imágenes encontradas en el `ImageRepository` con el fin de aplicar un criterio de selección. Por ejemplo, se buscará la etiqueta de la imagen más reciente cuya versión semántica sea mayor o igual a 0.1.0 (`>=0.1.0`).

Crear el fichero que contiene el recurso `ImagePolicy` y subir los cambios al repositorio:

```bash
flux create image policy echobot \
  --namespace=flux-system \
  --image-ref=echobot \
  --select-semver='>=0.1.0 <1.0.0' \
  --export > clusters/demo/automation/echobot-policy.yaml
```

<details>
  <summary>Resultado</summary>

  ```
  ---
  apiVersion: image.toolkit.fluxcd.io/v1alpha2
  kind: ImagePolicy
  metadata:
    name: echobot
    namespace: flux-system
  spec:
    imageRepositoryRef:
      name: echobot
    policy:
      semver:
        range: '>=0.1.0 <1.0.0'
  ```
</details>

```bash
{
  git add .
  git commit -m 'Add echobot image policy'
  git push origin main
}
```

Esperar a que se realice la sincronización con del repositorio o indíquele a Flux que realice el ciclo de reconciliación de manera inmediata:

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
  ✔ fetched revision main/208715e539dcc6f9b57bfeb3acd78f3195197d46
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/208715e539dcc6f9b57bfeb3acd78f3195197d46
  ```
</details>

Comprobar las imágenes detectadas que cumplen la política que hemos especificado (`>=0.1.0 <1.0.0`):

```bash
flux get image policy --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE  	NAME   	READY	MESSAGE                                                                  	LATEST IMAGE
  flux-system	echobot	True 	Latest image tag for 'ghcr.io/sngular/gitops-echobot' resolved to: v0.1.3	ghcr.io/sngular/gitops-echobot:v0.1.3
  ```
</details>

## Configurar el despliegue automático de la nueva etiqueta

Para que Flux pueda actualizar la imagen del servicio `echobot` es necesario:

* Añadir un marcador en el manifiesto de despliegue del servicio `echobot` para indicarle a Flux qué política debe aplicar al actualizar la imagen.
* Crear un recurso de tipo `ImageUpdateAutomation`. Flux lo utilizará para actualizar mediante un commit el manifiesto de despliegue (contenido en el `GitRepository`) con la nueva etiqueta de la imagen. Además podremos personalizar el mensaje del commit que hará Flux con información útil. 

### Añadir marcador para aplicar una política

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

Modifique el manifiesto de despliegue `echobot` y adiciónelo al repositorio:

```bash
cat <<EOF > ./clusters/demo/gitops-series/echobot-deployment.yaml
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
        image: ghcr.io/sngular/gitops-echobot:v0.1.0  # {"\$imagepolicy": "flux-system:echobot"}
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
  diff --git a/clusters/demo/gitops-series/echobot-deployment.yaml b/clusters/demo/gitops-series/echobot-deployment.yaml
  index 92dd1d6..f565077 100644
  --- a/clusters/demo/gitops-series/echobot-deployment.yaml
  +++ b/clusters/demo/gitops-series/echobot-deployment.yaml
  @@ -19,7 +19,7 @@ spec:
      spec:
        containers:
          - name: echobot
  -          image: ghcr.io/sngular/gitops-echobot:v0.1.0
  +          image: ghcr.io/sngular/gitops-echobot:v0.1.0  # {"$imagepolicy": "flux-system:echobot"}
            env:
              - name: CHARACTER
                value: "Esperando la actualización de imagen automágica!"
  ```
</details>

Incluir los cambios en el repositorio:

```bash
{
  git add .
  git commit -m 'Add marker to echobot image'
  git push origin main
}
```

### Configurar la automatización de la imagen

```bash
flux create image update echobot \
  --namespace=flux-system \
  --git-repo-ref=flux-system \
  --checkout-branch=main \
  --push-branch=main \
  --author-name=fluxbot \
  --author-email=fluxbot@gitops-series.com \
  --commit-template="{{ range .Updated.Images -}}
[demo] Automated image update **{{ \$.AutomationObject }}** to **{{ .Identifier }}**
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
{{ end -}}" \
  --export > clusters/demo/automation/echobot-automation.yaml
```

<details>
  <summary>Resultado</summary>

  ```
  ---
  apiVersion: image.toolkit.fluxcd.io/v1alpha2
  kind: ImageUpdateAutomation
  metadata:
    name: echobot
    namespace: flux-system
  spec:
    git:
      checkout:
        ref:
          branch: main
      commit:
        author:
          email: fluxbot@gitops-series.com
          name: fluxbot
        messageTemplate: |-
          {{ range .Updated.Images -}}
          [demo] Automated image update **{{ $.AutomationObject }}** to **{{ .Identifier }}**
          {{ end -}}
          Automation name: {{ .AutomationObject }}

          Files:
          {{ range $filename, $_ := .Updated.Files -}}
          - {{ $filename }}
          {{ end -}}

          Objects:
          {{ range $resource, $_ := .Updated.Objects -}}
          - {{ $resource.Kind }} {{ $resource.Name }}
          {{ end -}}

          Images:
          {{ range .Updated.Images -}}
          - {{.}}
          {{ end -}}
      push:
        branch: main
    interval: 1m0s
    sourceRef:
      kind: GitRepository
      name: flux-system
  ```
</details>

Incluir los cambios al repositorio:

```bash
{
  git add .
  git commit -m 'Add echobot automation'
  git push origin main
}
```

Esperar a que se realice la sincronización con del repositorio o indicarle a Flux que realice el ciclo de reconciliación de manera inmediata:

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
  ✔ fetched revision main/d502f4d9141fe2faeec555f3bae99a781155232f
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/d502f4d9141fe2faeec555f3bae99a781155232f
  ```
</details>

Comprobar las imágenes detectadas que cumplen la política que hemos especificado:

```bash
flux get image update --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE  	NAME   	READY	MESSAGE        	LAST RUN                 	SUSPENDED
  flux-system	echobot	True 	no updates made	2021-05-15T22:16:38+02:00	False
  ```
</details>

Comprobar que en el repositorio indicado en el recurso `GitRepository` habrá un nuevo commit de Flux con los datos de la actualización de la imagen.

Por último, comprobar que se ha actualizado la imagen a la versión más reciente del repositorio de imágenes y que esta coincide con la indicada en la política:

```bash
{
  flux get image policy --all-namespaces
  echo
  kubectl get deployment echobot -o jsonpath="{..image}" --namespace gitops-series
}
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE  	NAME   	READY	MESSAGE                                                                  	LATEST IMAGE
  flux-system	echobot	True 	Latest image tag for 'ghcr.io/sngular/gitops-echobot' resolved to: v0.1.3	ghcr.io/sngular/gitops-echobot:v0.1.3

  ghcr.io/sngular/gitops-echobot:v0.1.3
  ```
</details>

## (Opcional) Desintalar Flux

Si quieres desinstalar Flux puedes utilizar este comando:

```bash
flux uninstall --silent
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
