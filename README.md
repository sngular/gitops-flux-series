# GitOps Flux Series

A continuación se pueden consultar las guías de las diferentes secciones de la serie:

- [1.0 Descubriendo GitOps](1.0-descubriendo-gitops/README.md)
- [2.1 Instalación de Flux](2.1-instalacion-flux/README.md)
- [3.1 Fuente: GitRepository](3.1-fuente-gitrepository/README.md)
- [4.1 Secretos: Mozilla Sops](4.1-secretos-mozilla-sops/README.md)
- [5.1 Actualización automática de imágenes](5.1-actualizacion-automatica-imagenes/README.md)
- [6.1 Helm: Instalación y despliegues](6.1-helm-instalacion-despliegues/README.md)
- [6.2 Helm: Actualizaciones, Tests, Rollbacks y Dependencias](6.2-helm-actualizaciones-tests-rollbacks-dependencias/README.md)
- 7.1 Notification Controller
- 8.1 Monitorización

Nota: cada guía corresponde con un vídeo que podrá encontrar en la [lista de reproducción de YouTube](https://www.youtube.com/playlist?list=PLuQL-CB_D1E7gRzUGlchvvmGDF1rIiWkj).

## Requisitos

1) Disponer de un repositorio en Github, Gitlab o incluso puedes utilizar uno genérico.
2) Tener un cluster de Kubernetes que gestionar.

- Kubernetes en Cloud:
  - Google Cloud GKE: https://cloud.google.com/kubernetes-engine/
  - Amazon EKS: https://aws.amazon.com/eks/
  - Azure cuenta gratuita: https://azure.microsoft.com/es-es/services/kubernetes-service/
  - Civo: https://www.civo.com/
  - Digital Ocean: https://www.digitalocean.com/products/kubernetes/

- Kubernetes en local:
  - K3S: https://k3s.io/
  - K3D: https://k3d.io/
  - Minikube: https://minikube.sigs.k8s.io/docs/
  - Kind: https://kind.sigs.k8s.io/

Guías de aprovisionamiento de un cluster de Kubernetes:

- [Crear cluster de Kubernetes en CIVO](#crear-cluster-de-kubernetes-en-civo)
- [Crear cluster de Kubernetes con K3D](#crear-cluster-de-kubernetes-con-k3d)

## Crear cluster de Kubernetes con K3D

Instalar el binario k3d

```bash
wget -q -O - https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```

<details>
  <summary>Resultado</summary>

  ```
  Preparing to install k3d into /usr/local/bin
  k3d installed into /usr/local/bin/k3d
  Run 'k3d --help' to see what you can do with it.
  ```
</details>

Iniciar cluster

```bash
k3d cluster create demo
```

<details>
  <summary>Resultado</summary>

  ```
  INFO[0000] Prep: Network
  INFO[0000] Re-using existing network 'k3d-demo' (f24fb13aa1a6e642f1f8e1730fcb900c8295e6e39137b8dee216137872d89d76)
  INFO[0000] Created volume 'k3d-demo-images'
  INFO[0001] Creating node 'k3d-demo-server-0'
  INFO[0001] Creating LoadBalancer 'k3d-demo-serverlb'
  INFO[0001] Starting cluster 'demo'
  INFO[0001] Starting servers...
  INFO[0001] Starting Node 'k3d-demo-server-0'
  INFO[0009] Starting agents...
  INFO[0009] Starting helpers...
  INFO[0009] Starting Node 'k3d-demo-serverlb'
  INFO[0012] (Optional) Trying to get IP of the docker host and inject it into the cluster as 'host.k3d.internal' for easy access
  INFO[0017] Successfully added host record to /etc/hosts in 2/2 nodes and to the CoreDNS ConfigMap
  INFO[0017] Cluster 'demo' created successfully!
  INFO[0017] --kubeconfig-update-default=false --> sets --kubeconfig-switch-context=false
  INFO[0017] You can now use it like this:
  kubectl config use-context k3d-demo
  kubectl cluster-info
  ```
</details>

Listar nodos

```bash
kubectl get nodes
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                STATUS   ROLES                  AGE     VERSION
  k3d-demo-server-0   Ready    control-plane,master   4m31s   v1.21.0+k3s1
  ```
</details>

Listar pods:

```bash
kubectl get pods --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
  kube-system   helm-install-traefik-crd-5n55c            0/1     Completed   0          5m52s
  kube-system   metrics-server-86cbb8457f-bp88w           1/1     Running     0          5m52s
  kube-system   coredns-7448499f4d-sj9dt                  1/1     Running     0          5m52s
  kube-system   local-path-provisioner-5ff76fc89d-ndlwm   1/1     Running     0          5m52s
  kube-system   helm-install-traefik-rgc9f                0/1     Completed   0          5m52s
  kube-system   svclb-traefik-bhqvb                       2/2     Running     0          2m
  kube-system   traefik-97b44b794-5gf89                   1/1     Running     0          119s
  ```
</details>

## Crear cluster de Kubernetes en CIVO

Registrarse en [Civo](https://www.civo.com/)

Instalar el binario de civo:

```bash
{
  curl -sL https://civo.com/get | sh
  sudo mv /tmp/civo /usr/local/bin/civo
}
```

<details>
  <summary>Resultado</summary>

  ```
  /usr/bin/curl
  Finding latest version from GitHub
  0.7.22
  Downloading package https://github.com/civo/cli/releases/download/v0.7.22/civo-0.7.22-linux-amd64.tar.gz to /tmp/civo-0.7.22-linux-amd64.tar.gz
  Download complete.

  ============================================================
    The script was run as a user who is unable to write
    to /usr/local/bin. To complete the installation the
    following commands may need to be run manually.
  ============================================================

  sudo mv /tmp/civo /usr/local/bin/civo

  [sudo] password:
  ```
</details>

Configurar el API key, la clave se encuentra en la sección [security](https://www.civo.com/account/security) de la cuenta de Civo:

```bash
{
  civo apikey save gitops-flux <API_KEY>
  civo apikey ls
}
```

<details>
  <summary>Resultado</summary>

  ```
  Saved the API Key gitops-flux as <API_KEY>

  +-------------+---------+
  | Name        | Default |
  +-------------+---------+
  | gitops-flux | <=====  |
  +-------------+---------+
  ```
</details>

Crear el cluster eligiendo el nombre, la región y el tamaño

```bash
civo kubernetes create demo-flux \
  --size "g3.k3s.large" \
  --region "LON1" \
  --save \
  --merge \
  --switch \
  --wait \
  --yes
```

<details>
  <summary>Resultado</summary>

  ```
  Creating a 3 node k3s cluster of g3.k3s.large instances called demo-flux... \

  Access your cluster with:
  kubectl get node
  The cluster demo-flux (992893cd-7c33-4490-933b-1576b9ad9462) has been created in 6 min 55 sec
  ```
</details>

Listar nodos:

```bash
kubectl get node
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                                    STATUS   ROLES    AGE   VERSION
  k3s-demo-flux-7743ec26-node-pool-2b6c   Ready    <none>   15m   v1.20.2+k3s1
  k3s-demo-flux-7743ec26-node-pool-b232   Ready    <none>   15m   v1.20.2+k3s1
  k3s-demo-flux-7743ec26-node-pool-cc1a   Ready    <none>   15m   v1.20.2+k3s1
  ```
</details>

Listar pods:

```bash
kubectl get pods --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
  kube-system   helm-install-traefik-9qfsx                0/1     Completed   0          21m
  kube-system   local-path-provisioner-7c458769fb-qd6k5   1/1     Running     0          21m
  kube-system   metrics-server-86cbb8457f-vps2w           1/1     Running     0          21m
  kube-system   svclb-traefik-zg85k                       2/2     Running     0          16m
  kube-system   svclb-traefik-w57dg                       2/2     Running     0          16m
  kube-system   traefik-6f9cbd9bd4-f7xj7                  1/1     Running     0          16m
  kube-system   svclb-traefik-blqn2                       2/2     Running     0          16m
  kube-system   coredns-854c77959c-s82cn                  1/1     Running     0          21m
  ```
</details>

Mostrar cluster:

```bash
civo kubernetes show demo-flux
```

<details>
  <summary>Resultado</summary>

  ```
            ID : 992893cd-7c33-4490-933b-1576b9ad9462
          Name : demo-flux
        Region : LON1
        Nodes : 3
          Size : g3.k3s.large
        Status : ACTIVE
      Version : 1.20.0-k3s1
  API Endpoint : https://74.220.18.170:6443
  External IP : 74.220.18.170
  DNS A record : 992893cd-7c33-4490-933b-1576b9ad9462.k8s.civo.com

  Pool (648b29):
  +---------------------------------------+----+--------+------+-----------+------+----------+
  | Name                                  | IP | Status | Size | Cpu Cores | Ram  | SSD disk |
  +---------------------------------------+----+--------+------+-----------+------+----------+
  | k3s-demo-flux-7743ec26-node-pool-2b6c |    | ACTIVE |      |         4 | 8192 |       15 |
  | k3s-demo-flux-7743ec26-node-pool-b232 |    | ACTIVE |      |         4 | 8192 |       15 |
  | k3s-demo-flux-7743ec26-node-pool-cc1a |    | ACTIVE |      |         4 | 8192 |       15 |
  +---------------------------------------+----+--------+------+-----------+------+----------+
  Labels:
  kubernetes.civo.com/node-pool=648b2926-b2af-4196-9c43-c3791eb29122
  kubernetes.civo.com/node-size=g3.k3s.large
  ```
</details>

Eliminar cluster:

```bash
civo kubernetes remove demo-flux --yes
```

<details>
  <summary>Resultado</summary>

  ```
  The Kubernetes cluster (demo-flux) has been deleted
  ```
</details>

## Referencias

- Documentación oficial de Flux: https://fluxcd.io/docs/

