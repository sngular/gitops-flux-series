# 4.1 Secretos: Mozilla Sops

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >= 0.13.2
* Tener instalado gnupg >=2.2.24
* Tener instalado sops >=3.7.1

## Comprobar que el cluster cumple los requisitos

```bash
flux check --pre
```

## Exportar token de GitHub

```bash
export GITHUB_TOKEN=<your-token>
```

## Instalar Flux

```bash
flux bootstrap github \
  --owner=sngular \
  --repository=gitops-flux-series-demo \
  --branch=main \
  --private=false \
  --path=./cluster/namespaces
```

## Clonar repositorio creado

```bash
{
  git clone git@github.com:sngular/gitops-flux-series-demo.git
  cd gitops-flux-series-demo
  tree
}
```

## Copiar los manifiestos del namespace gitops-series

```bash
{
  cp -r ../gitops-flux-series/4.1-secretos-mozilla-sops/manifests/gitops-series cluster/namespaces
  tree
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

## Generar la llave GPG

```bash
export KEY_NAME="demo.gitops-series.io"
export KEY_COMMENT="flux secrets"


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

```bash
gpg --list-secret-keys "${KEY_NAME}"

sec   rsa4096 2021-05-11 [SCEA]
      551A4D3A98D5CABC697373CB42F867BF0DDB91CF
```

Almacenar la huella digital de la clave como una variable de entorno:

```bash
export KEY_FP=551A4D3A98D5CABC697373CB42F867BF0DDB91CF
```

## Adicionar la llave privada al cluster

Crear el secreto de Kubernetes `sops-gpg` utilizando las claves públicas y privadas almacenadas en GPG

```bash
gpg --export-secret-keys --armor "${KEY_FP}" |
kubectl create secret generic sops-gpg \
  --namespace=flux-system \
  --from-file=sops.asc=/dev/stdin
```

Listar el secreto creado

```bash
kubectl --namespace flux-system get secrets sops-gpg
```

Exportar a un fichero la llave pública y adicionarla al repositorio

```bash
{
  gpg --export --armor "${KEY_FP}" > ./cluster/.sops.pub.asc
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

Listar la llave pública

```bash
gpg --list-public-keys  "${KEY_NAME}"
```

## Habilitar descifrado con sops

```bash
{
  cp -r ../gitops-flux-series/4.1-secretos-mozilla-sops/manifests/flux-system/ cluster/namespaces/flux-system/
  git add .
  git commit -m 'Enable decryption sops in flux-system kustomization'
  git push origin main
}
```

Acelerar el ciclo de reconciliación

```bash
{
  flux reconcile source git flux-system
  flux reconcile kustomization flux-system
}
```

## Configurar encriptación con sops

```bash
cat <<EOF > ./cluster/.sops.yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    pgp: ${KEY_FP}
EOF
```

## Crear secreto encriptado al repositorio

```bash
kubectl --namespace gitops-series create secret generic secret-text \
  --from-literal=hidden-text="mensaje confidencial" \
  --dry-run=client \
  -o yaml > cluster/namespaces/gitops-series/secret-text.yaml
```

Encriptar el secreto con sops

```bash
sops --config cluster/.sops.yaml \
  --encrypt \
  --in-place cluster/namespaces/gitops-series/secret-text.yaml
```

Adicionar el secreto encriptado al repositorio

```bash
{
  git add .
  git commit -m 'Add secret text to gitops-series namespace'
  git push origin main
}
```

Acelerar el ciclo de reconciliación

```bash
{
  flux reconcile source git flux-system
  flux reconcile kustomization flux-system
}
```

Listar los secretos del namespace `gitops-series`:

```bash
kubectl -n gitops-series get secrets
```

Mostar el mensaje oculto del secreto

```bash
kubectl --namespace gitops-series \
get secrets secret-text \
-o jsonpath='{.data.hidden-text}' | base64 --decode
```

## Utilizar el mensaje oculto

```bash
{
  cp -r ../gitops-flux-series/4.1-secretos-mozilla-sops/manifests/secret-text/ cluster/namespaces/gitops-series/
  git add .
  git commit -m 'Use hidden-text in echobot deployment'
  git push origin main
}
```

Acelerar el ciclo de reconciliación

```bash
{
  flux reconcile source git flux-system
  flux reconcile kustomization flux-system
}
```

Mostrar los logs:

```bash
kubectl --namespace gitops-series logs \
  --selector app=echobot \
  --follow
```