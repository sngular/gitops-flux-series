# 4.1 Secretos: Mozilla Sops

En esta sección se mostrará cómo Flux gestiona los secretos de Kubernetes. Aprenderá a gestionar la información sensible en equipo con herramientas efectivas e innovadoras como Mozilla Sops.

Vídeo de la explicación y la demo completa en este [enlace](https://www.youtube.com/watch?v=cThJJMgGSKM&list=PLuQL-CB_D1E7gRzUGlchvvmGDF1rIiWkj&index=4).

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >= 0.13.2
* Tener instalado gnupg >=2.2.24
* Tener instalado sops >=3.7.1
* Tener instalado jq >=1.6

## Exportar token de GitHub

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

## Instalar Flux en el cluster

Utilice el comando `bootstrap` para instalar los componentes de Flux en el cluster, crear el repositorio en GitHub y mucho más:

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
  ✔ committed sync manifests to "main" ("b48c46ec7e8712f064fafadd222a4b781b879570")
  ► pushing component manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► installing components in "flux-system" namespace
  ✔ installed components
  ✔ reconciled components
  ► determining if source secret "flux-system/flux-system" exists
  ► generating source secret
  ✔ public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDfLYOedkqvUlPn7vAkyLwaOpUyoGdigVWytV3ztxEYQ/7k1jN6ZTzVvjt+P8S3XO6Rj02NuZdTMuX+GyNlD8ZsJytK0hdpSOO+1nbMJZZ/dCvETKd6usmzqgZ1MinO9pKEiEtddeKYdisZfQIJ6fnVMUQPSfSg2g38Tl77cFnQcZl+33cVnztlabo7T15NRQH1sE67PXS5svofu3oE+ksmhCck2o5GTpucBQbWfpgKblCxcDP6RAZ/DQg8vNowDIHlCbAEnWABMOlj23GH8fkbQX1p/emNxhDSshhlD0Q6avZ20SL7HwR+Cha3tB6bn0NMtpcfNcZi5ks8lN8m6OZv
  ✔ configured deploy key "flux-system-main-flux-system-./clusters/demo" for "https://github.com/sngular/gitops-flux-series-demo"
  ► applying source secret "flux-system/flux-system"
  ✔ reconciled source secret
  ► generating sync manifests
  ✔ generated sync manifests
  ✔ committed sync manifests to "main" ("d04db5d7fbeb93ebfdb954657f88fc082875e2a9")
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

## Crear los manifiestos del namespace gitops-series

Crear carpeta `gitops-series`:

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

Crear el fichero del deployment:

```bash
cat <<EOF > ./clusters/demo/gitops-series/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echobot
  namespace: gitops-series
  labels:
    app: echobot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echobot
  template:
    metadata:
      labels:
        app: echobot
    spec:
      containers:
        - name: message
          image: ghcr.io/sngular/gitops-echobot:v0.1.0
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              cpu: 10m
              memory: 30Mi
            limits:
              cpu: 10m
              memory: 30Mi
EOF
```

Incluya los ficheros creados en el control de versiones:

```bash
{
  git add .
  git commit -m 'Add gitops series manifests'
  git push origin main
}
```

Observar la creación del pod:

```bash
watch -n1 kubectl get pods \
  --namespace gitops-series
```

> Si no desea esperar por el ciclo de reconciliación utilice el siguiente comando:

```bash
flux reconcile kustomization flux-system --with-source
```

Mostrar los logs:

```bash
kubectl --namespace gitops-series logs \
  --selector app=echobot \
  --follow
```

<details>
  <summary>Resultado</summary>

  ```bash
  hostname: echobot-d997b8cf5-pzng4 - gitops flux series
  hostname: echobot-d997b8cf5-pzng4 - gitops flux series
  hostname: echobot-d997b8cf5-pzng4 - gitops flux series
  hostname: echobot-d997b8cf5-pzng4 - gitops flux series
  hostname: echobot-d997b8cf5-pzng4 - gitops flux series
  ```
</details>

## Generar la clave GPG

```bash
{
  export KEY_NAME="demo.gitops-series.io"
  export KEY_COMMENT="flux secrets"
}
```

```bash
gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: ${KEY_COMMENT}
Name-Real: ${KEY_NAME}
EOF
```

Recuperar la huella digital de la clave GPG:

> La huella digital se encuentra en la segunda línea de la salida en consola.

```bash
gpg --list-secret-keys "${KEY_NAME}"
```
<details>
  <summary>Resultado</summary>

  ```bash
  gpg: checking the trustdb
  gpg: marginals needed: 3  completes needed: 1  trust model: pgp
  gpg: depth: 0  valid:   3  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 3u
  gpg: next trustdb check due at 2024-06-30
  sec   rsa4096 2021-05-13 [SCEA]
        5999BA5E167E0A7E60B4F66D13767E9916D79699
  uid           [ultimate] demo.gitops-series.io (flux secrets)
  ssb   rsa4096 2021-05-13 [SEA]
  ```
</details>

Almacenar la huella digital de la clave como una variable de entorno:

```bash
export KEY_FP=$(gpg --list-secret-keys "${KEY_NAME}" | grep --extended-regexp --only-matching '^\s.*$' | tr -d ' ')
```

> Listar clave con detalles, filtrar la huella digital y exportar la variablede entorno.

Nota: En el ejemplo la huella digital es `5999BA5E167E0A7E60B4F66D13767E9916D79699`.

## Agregar la clave privada al cluster

Crear el secreto de Kubernetes `sops-gpg` utilizando las claves públicas y privadas almacenadas en GPG:

```bash
gpg --export-secret-keys --armor "${KEY_FP}" |
kubectl create secret generic sops-gpg \
  --namespace=flux-system \
  --from-file=sops.asc=/dev/stdin
```

> Este objeto se crea directamente en el cluster de forma excepcional para no almacenar la clave privada en el repositorio.

Listar el secreto creado:

```bash
kubectl --namespace flux-system get secrets sops-gpg
```

<details>
  <summary>Resultado</summary>

  ```bash
  NAME       TYPE     DATA   AGE
  sops-gpg   Opaque   1      108s
  ```
</details>

Exportar a un fichero la clave pública y añadirla al repositorio de código:

```bash
{
  gpg --export --armor "${KEY_FP}" > ./clusters/demo/.sops.pub.asc
  git add .
  git commit -m 'Add sops public key'
  git push origin main
}
```

## Eliminar la clave privada

```bash
gpg --delete-secret-keys "${KEY_FP}"
```

Listar la clave privada

```bash
gpg --list-secret-keys  "${KEY_NAME}"
```

<details>
  <summary>Resultado</summary>

  ```bash
  gpg: error reading key: No secret key
  ```
</details>

Listar la clave pública

```bash
gpg --list-public-keys  "${KEY_NAME}"
```

<details>
  <summary>Resultado</summary>

  ```bash
  pub   rsa4096 2021-05-12 [SCEA]
        5FBAF292C87E424E2ADA880E909D8D2E11EC9EAF
  uid           [ultimate] demo.gitops-series.io (flux secrets)
  sub   rsa4096 2021-05-12 [SEA]
  ```
</details>

## Habilitar descifrado con sops

Crear el fichero `gotk-patches.yaml` para modificar los ficheros de instalación de Flux:

```bash
cat <<EOF > ./clusters/demo/flux-system/gotk-patches.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
EOF
```

Actualizar el fichero `kustomization.yaml` para que utilice el fichero `gotk-patches.yaml` durante el próximo ciclo de reconciliación:

```bash
cat <<EOF > ./clusters/demo/flux-system/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- gotk-components.yaml
- gotk-sync.yaml
patchesStrategicMerge:
- gotk-patches.yaml
EOF
```

Añada los cambios en el control de versiones:

```bash
{
  git add .
  git commit -m 'Enable decryption sops in flux-system kustomization'
  git push origin main
}
```

Mostrar los logs de los componentes de Flux:

```bash
flux logs --name=flux-system --follow
```

<details>
  <summary>Resultado</summary>

  ```bash
  2021-06-06T21:43:44.676Z info Kustomization/flux-system.flux-system - Discarding event, no alerts found for the involved object
  2021-06-06T21:43:45.576Z info Kustomization/flux-system.flux-system - Discarding event, no alerts found for the involved object
  2021-06-06T21:43:51.862Z info Kustomization/flux-system.flux-system - Kustomization applied in 2.4085245s
  2021-06-06T21:43:52.035Z info Kustomization/flux-system.flux-system - Reconciliation finished in 6.4570337s, next run in 10m0s
  2021-06-06T21:45:04.901Z info Kustomization/flux-system.flux-system - Kustomization applied in 1.7873658s
  2021-06-06T21:45:04.956Z info Kustomization/flux-system.flux-system - Reconciliation finished in 3.2329504s, next run in 10m0s
  ```
</details>

Acelerar el ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

Comprobar el estado final del objeto Kustomization `flux-system`:

```bash
kubectl get kustomizations.kustomize.toolkit.fluxcd.io flux-system \
  --namespace flux-system \
  -o jsonpath='{.spec}' | jq
```

## Configurar la encriptación con sops

El fichero `.sops.yaml` permite definir la forma de encriptar la información.

```bash
cat <<EOF > ./clusters/demo/.sops.yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    pgp: ${KEY_FP}
EOF
```

## Crear secreto encriptado al repositorio

Crear un secreto de k8s en el fichero `secret-text.yaml`:

```bash
kubectl create secret generic secret-text \
  --namespace gitops-series \
  --from-literal=hidden-text="mensaje confidencial" \
  --dry-run=client \
  -o yaml > clusters/demo/gitops-series/secret-text.yaml
```

<details>
  <summary>Resultado</summary>

  ```yaml
  apiVersion: v1
  data:
    hidden-text: bWVuc2FqZSBjb25maWRlbmNpYWw=
  kind: Secret
  metadata:
    creationTimestamp: null
    name: secret-text
    namespace: gitops-series
  ```
</details>

Encriptar el secreto con sops:

```bash
sops --config clusters/demo/.sops.yaml \
  --encrypt \
  --in-place clusters/demo/gitops-series/secret-text.yaml
```

<details>
  <summary>Resultado</summary>

  ```yaml
  apiVersion: v1
  data:
      hidden-text: ENC[AES256_GCM,data:a6PP5TXhYiyolFz3BJxnUPV352NgsocBpOIoJg==,iv:e3j1lapS4kV2nEROYXPkgVmOmtX0MNDE0rqcoabpNd0=,tag:PX+079lT/xYh2ecIw1+dUg==,typ
  e:str]
  kind: Secret
  metadata:
      creationTimestamp: null
      name: secret-text
      namespace: gitops-series
  sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age: []
    lastmodified: "2021-06-06T21:59:18Z"
    mac: ENC[AES256_GCM,data:BCywKngSZcaot0zhUCdmhpjfe3Ww8kbUzXHK7n+8Tp1cvsYGnbRUxEX94cVW9qOAAOJdFMomuOj3UvKrDAg9tlKvzpc+XL5K5aiL4teMDa4ewcK0+XrtcwD4r4zJT5jNj
rUBJ2EK81lVhBqQymzdArgK5NuQ3+ybdBjVduyPe9A=,iv:VlEzRBkADpirBNxvdslcfDTkviSyjRR6ZEanTL8I2Tc=,tag:ZhBfvGod+LyFDfZSh8JaZg==,type:str]
    pgp:
        - created_at: "2021-06-06T21:59:17Z"
          enc: |
            -----BEGIN PGP MESSAGE-----

            hQIMA17x+O1VhqU3ARAAjp1sddeq2LezbbCs5sz6InEe2RCaWDZPRjyAQi5kjxl6
            Ar33zQQ18XS9NAqxjcegfEXYqEkFSkH9Oc5kxpXhMLZP7031fTdWiAMemG+G0hba
            LtQnhGgszj+/s68gfacH+RccxjxB8s/u7O3FihtWRgdySxQ+dDNnWEH5PT15TnpG
            hfIgy6yULoBMYFNQ/Zmm6rJisNTwMPN3lKCb8Gll2ue1MhCoshMI70D5JTYPXY06
            0pYw5rWywSLwd/XmiuzbsPn4Kwfg5kVpVXAyOVlC6JJA05xU11apdAjsRyOnd8Hu
            hPFVwMwT4y+AJbSXD4ow/LuL6HS2HD0B02qra8mBj3772Ki+VDrwH2kEGPSE/MQs
            9op53EL0UvVMxHgUoLaP7wOaCMIF9IQgI7oGWDIP5oO8+aId5g0LtNxA6MpZqshb
            bYb+yPnF9WECF+AKVTbdf4mMudub2SNfylnOpbhTef5yOCM+v0JAQjxKACykL9gi
            SUpXpMoc5c2X1nkf5lrJuJHHMUjEIzq070AM85fu3Un1btscfexC3V6h4BAuxfH2
            yuo7TUop46IPGXZX9tQZMWVVWvDuS1crUEVWLlux3c1otgmDiN3V0IDwis2EaDBM
            RqOlIkLRkTFovRZS+a9zlDpIWGawMEAGJNLitfjOc6OVdQpr1eJ5usVKgIvdQzHS
            XAHdw2aX04/8Yoykr6cbGBKDG/90fAo4qKzZLC2khQn3lBoIpaSqbIJNlfnyPAQd
            o5cdIlaF0Hpr51KdDPlHqNEWsRbU3rxQOKds201LVxs5B/uqoHw3gMOwC390
            =/mVp
            -----END PGP MESSAGE-----
          fp: A7070BDBD09AC244EA37D0152E178A90D9FA8086
    encrypted_regex: ^(data|stringData)$
    version: 3.7.1
  ```
</details>

Agrear el secreto encriptado al repositorio:

```bash
{
  git add .
  git commit -m 'Add secret text to gitops-series namespace'
  git push origin main
}
```

Listar los secretos del namespace `gitops-series`:

```bash
watch kubectl get secrets \
  --namespace=gitops-series
```

<details>
  <summary>Resultado</summary>

  ```bash
  NAME                  TYPE                                  DATA   AGE
  default-token-wvt5m   kubernetes.io/service-account-token   3      26m
  secret-text           Opaque                                1      69s
  ```
</details>

> Acelerar el ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

Mostar el mensaje oculto del secreto:

```bash
kubectl get secrets secret-text \
  --namespace gitops-series \
  -o jsonpath='{.data.hidden-text}' | base64 --decode
```

<details>
  <summary>Resultado</summary>

  ```bash
  mensaje confidencial%
  ```
</details>

## Utilizar el mensaje oculto del secreto

Actualizar el fichero `deployment.yaml`:

```bash
cat <<EOF > ./clusters/demo/gitops-series/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echobot
  namespace: gitops-series
  labels:
    app: echobot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echobot
  template:
    metadata:
      labels:
        app: echobot
    spec:
      containers:
        - name: message
          image: ghcr.io/sngular/gitops-echobot:v0.1.0
          imagePullPolicy: IfNotPresent
          env:
            - name: CHARACTER
              valueFrom:
                secretKeyRef:
                  name: secret-text   # se utiliza el secret creado
                  key: hidden-text
          resources:
            requests:
              cpu: 10m
              memory: 30Mi
            limits:
              cpu: 10m
              memory: 30Mi
EOF
```

<details>
  <summary>Git diff</summary>

  ```
  diff --git a/clusters/demo/gitops-series/deployment.yaml b/clusters/demo/gitops-series/deployment.yaml
  index ef47ef9..6c25fb4 100644
  --- a/clusters/demo/gitops-series/deployment.yaml
  +++ b/clusters/demo/gitops-series/deployment.yaml
  @@ -19,6 +19,12 @@ spec:
           - name: message
             image: ghcr.io/sngular/gitops-echobot:v0.1.0
             imagePullPolicy: IfNotPresent
  +          env:
  +            - name: CHARACTER
  +              valueFrom:
  +                secretKeyRef:
  +                  name: secret-text   # se utiliza el secret creado
  +                  key: hidden-text
             resources:
               requests:
                 cpu: 10m
  ```
</details>

Establecer los cambios en el repositorio de código:

```bash
{
  git add .
  git commit -m 'Use hidden-text in echobot deployment'
  git push origin main
}
```

```bash
watch kubectl get pods \
  --namespace=gitops-series
```

> Acelerar el ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

Mostrar los logs:

```bash
kubectl logs \
  --namespace gitops-series \
  --selector app=echobot \
  --follow
```

<details>
  <summary>Resultado</summary>

  ```bash
  hostname: echobot-7478f9f457-thrst - mensaje confidencial
  hostname: echobot-7478f9f457-thrst - mensaje confidencial
  hostname: echobot-7478f9f457-thrst - mensaje confidencial
  hostname: echobot-7478f9f457-thrst - mensaje confidencial
  ```
</details>

## (Opcional) Desintalar Flux

Si desea desinstalar Flux puede utilizar el siguiente comando:

```bash
flux uninstall --silent
```

> Compruebe que el repositorio en GitHub no ha sido eliminado.

<details>
  <summary>Resultado</summary>

  ```
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
