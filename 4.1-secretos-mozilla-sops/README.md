# 4.1 Secretos: Mozilla Sops

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >= 0.13.2
* Tener instalado gnupg >=2.2.24
* Tener instalado sops >=3.7.1

## Estado del cluster

Comprobar que se cumplen los requisitos previos necesarios para instalar Flux2.

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

## Instalar Flux

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
  ✔ committed sync manifests to "main" ("a5d221a41afb59e37edc3cba353e7b1f31dffaa4")
  ► pushing component manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► installing components in "flux-system" namespace
  ✔ installed components
  ✔ reconciled components
  ► determining if source secret "flux-system/flux-system" exists
  ► generating source secret
  ✔ public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDIRvouN6owHS6z7/gYimHX0ZsFBMzmFVs0OLMD9XZwilETOaCPxAbNcvuMlvJpqNiRDjZ+zeI/iakCpNg88QNI69PAJ/zU2SnAa6cR9uisI3jU9VqSCBY78hN4uprMhwSwANcFq/lPg3umYqtlKOXuOBbMBZab8GVQgwl93Tg4h8/Xe3OnxFOP2YqFgNA4vWgJC9F9cWL7K+6ZW0dZcixNZo0Wep36DNRJczYctsjjU+/UBd4OTWAG++KXZwLNj/Bvcl37+7yhbGQZ6gVYFZWzGibzNUznIbak9FslvjQOijs7IGTnNmwrSo7RheQvMieJTHT8TC8J2NuozK7VTV+3
  ✔ configured deploy key "flux-system-main-flux-system-./clusters/demo" for "https://github.com/sngular/gitops-flux-series-demo"
  ► applying source secret "flux-system/flux-system"
  ✔ reconciled source secret
  ► generating sync manifests
  ✔ generated sync manifests
  ✔ committed sync manifests to "main" ("5f7f0d94f330ac65bd92d0b0a3a14c7fec8eb64d")
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

## Clonar repositorio creado

```bash
{
  git clone git@github.com:$GITHUB_USER/gitops-flux-series-demo.git
  cd gitops-flux-series-demo
  tree
}
```

<details>
  <summary>Resultado</summary>

  ```bash
  Cloning into 'gitops-flux-series-demo'...
  remote: Enumerating objects: 13, done.
  remote: Counting objects: 100% (13/13), done.
  remote: Compressing objects: 100% (6/6), done.
  remote: Total 13 (delta 0), reused 13 (delta 0), pack-reused 0
  Receiving objects: 100% (13/13), 17.43 KiB | 3.49 MiB/s, done.
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

## Crear objetos del namespace gitops-series

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
  git commit -m 'Add gitops series namespace'
  git push origin main
}
```

Observar la creación del pod:

```bash
watch -n1 kubectl get pods --namespace gitops-series
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

## Generar la llave GPG

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
# Listar clave con detalles, filtrar la huella digital y exportar la variablede entorno.
export KEY_FP=$(gpg --list-secret-keys "${KEY_NAME}" | grep --extended-regexp --only-matching '^\s.*$' | tr --delete ' ')
```

Nota: En el ejemplo la huella digital es `5999BA5E167E0A7E60B4F66D13767E9916D79699`.

## Adicionar la llave privada al cluster

Crear el secreto de Kubernetes `sops-gpg` utilizando las claves públicas y privadas almacenadas en GPG:

```bash
gpg --export-secret-keys --armor "${KEY_FP}" |
kubectl create secret generic sops-gpg \
  --namespace=flux-system \
  --from-file=sops.asc=/dev/stdin
```

> Este objeto se crea directamente en el cluster de forma excepcional para no almacenar la llave privada en el repositorio.

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

Exportar a un fichero la llave pública y adicionarla al repositorio de código:

```bash
{
  gpg --export --armor "${KEY_FP}" > ./clusters/demo/.sops.pub.asc
  git add .
  git commit -m 'Add sops public key'
  git push origin main
}
```

## Eliminar la llave privada

```bash
gpg --delete-secret-keys "${KEY_FP}"
```

Listar la llave privada

```bash
gpg --list-secret-keys  "${KEY_NAME}"
```

<details>
  <summary>Resultado</summary>

  ```bash
  gpg: error reading key: No secret key
  ```
</details>

Listar la llave pública

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

Actualice el objeto Kustomization `flux-system`:

```bash
cat <<EOF > ./clusters/demo/flux-system/gotk-sync.yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: ssh://git@github.com/sngular/gitops-flux-series-demo
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/demo
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  validation: client
  decryption:                 # configuración sops
    provider: sops
    secretRef:
      name: sops-gpg
EOF
```

<details>
  <summary>Git diff</summary>

  ```
  diff --git a/clusters/demo/flux-system/gotk-sync.yaml b/clusters/demo/flux-system/gotk-sync.yaml
  index 356002c..e798dce 100644
  --- a/clusters/demo/flux-system/gotk-sync.yaml
  +++ b/clusters/demo/flux-system/gotk-sync.yaml
  @@ -25,3 +25,7 @@ spec:
       kind: GitRepository
       name: flux-system
     validation: client
  +  decryption:                 # configuración sops
  +    provider: sops
  +    secretRef:
  +      name: sops-gpg
  ```
</details>

Adicione los cambios en el control de versiones:

```bash
{
  git add .
  git commit -m 'Enable decryption sops in flux-system kustomization'
  git push origin main
}
```

Acelerar el ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

<details>
  <summary>Resultado</summary>

  ```bash
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/3697781dd9efe5f5454c3fc1973ab73f3ec4b9b2
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/3697781dd9efe5f5454c3fc1973ab73f3ec4b9b2
  ```
</details>

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

Encriptar el secreto con sops:

```bash
sops --config clusters/demo/.sops.yaml \
  --encrypt \
  --in-place clusters/demo/gitops-series/secret-text.yaml
```

Adicionar el secreto encriptado al repositorio:

```bash
{
  git add .
  git commit -m 'Add secret text to gitops-series namespace'
  git push origin main
}
```

Acelerar el ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

<details>
  <summary>Resultado</summary>

  ```bash
  ► annotating GitRepository flux-system in flux-system namespace
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/33c524cb7fb41b3128e26049aa36676cbe05cc79
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/33c524cb7fb41b3128e26049aa36676cbe05cc79
  ```
</details>

Listar los secretos del namespace `gitops-series`:

```bash
kubectl -n gitops-series get secrets
```

<details>
  <summary>Resultado</summary>

  ```bash
  NAME                  TYPE                                  DATA   AGE
  default-token-wvt5m   kubernetes.io/service-account-token   3      26m
  secret-text           Opaque                                1      69s
  ```
</details>

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

Acelerar el ciclo de reconciliación:

```bash
flux reconcile kustomization flux-system --with-source
```

<details>
  <summary>Resultado</summary>

  ```bash
  ► annotating GitRepository flux-system in flux-system namespace
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/38d460552d277e7aebcb6857039e62a49ad35783
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/38d460552d277e7aebcb6857039e62a49ad35783
  ```
</details>

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
