# GitOps Flux Series

A continuación se pueden consultar las guías de las diferentes secciones de la serie:

- [1.0 Descubriendo GitOps](1.0-descubriendo-gitops/README.md)
- [2.1 Instalación de Flux](2.1-instalacion-flux/README.md)
- [3.1 Fuente: GitRepository](3.1-fuente-gitrepository/README.md)
- [4.1 Secretos: Mozilla Sops](4.1-secretos-mozilla-sops/README.md)
- [5.1 Actualización automática de imágenes](5.1-actualizacion-automatica-imagenes/README.md)
- 6.1 Helm Controller: Integración con Helm
- 6.2 Helm Controller: Actualización de imágenes y rollbacks
- 6.3 Helm Controller: Testing
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

Listar pods

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

## Referencias

- Documentación oficial de Flux: https://fluxcd.io/docs/

