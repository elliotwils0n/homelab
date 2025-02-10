# worker

## Configuration
### Cluster
- Install [k3s](https://docs.k3s.io/quick-start) without traefik
  ```shell
  curl -sfL https://get.k3s.io | sh -s - --disable=traefik
  ```
- Init configuration for non-root user
  ```shell
  mkdir -p ~/.kube
  sudo k3s kubectl config view --raw > ~/.kube/config
  chmod 600 ~/.kube/config
  echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
  ```
- __if__ something goes wrong - [uninstall instructions](https://docs.k3s.io/installation/uninstall)
  ```shell
  /usr/local/bin/k3s-uninstall.sh
  ```
- [Optional] setup additional tls-san
  <details>
  <summary>k3s config</summary>

  ```
  sudo mkdir -p /etc/rancher/k3s/config.yaml.d
  sudo tee /etc/rancher/k3s/config.yaml.d/tls-san.yaml > /dev/null <<EOF
  tls-san+:
    - 192.168.0.211
    - vm1.home.arpa
  EOF
  sudo systemctl stop k3s
  sudo k3s certificate rotate
  sudo systemctl start k3s
  ```
  </details>
- to resolve host for homelab without registered domain add records to coredns
  <details>
  <summary>records</summary>

  ```
  192.168.0.200 forgejo.home.arpa
  192.168.0.200 registry.home.arpa
  192.168.0.200 tempo.home.arpa
  192.168.0.200 loki.home.arpa
  192.168.0.200 prometheus.home.arpa
  ```
  </details>

  ```shell
  kubectl -n kube-system edit cm coredns
  kubectl -n kube-system rollout restart deploy coredns
  ```
- Move initial cluster manifests to [k3s AddOns directory](https://docs.k3s.io/installation/packaged-components).
  ```
  sudo cp configuration/* /var/lib/rancher/k3s/server/manifests/.
  ```

### Post bootstrap
- Data to create [declarative cluster manifest](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters)
  in Argo CD can be found in argocd-manager-token secret
  ```shell
  kubectl -n kube-system get secret argocd-manager-token -o yaml
  ```
  With that create a secret and apply it in Argo CD project configuration
  (i.e. [cluster secret](../cicd/services/argocd/projects/example/cluster-secret.yaml))
- [Optional] create namespace for example project with registry credentials
  ```shell
  kubectl create namespace example
  kubectl label namespace example istio-injection=enabled
  kubectl create secret docker-registry image-registry \
    --docker-server=registry.home.arpa \
    --docker-username=example-bot \
    --docker-password=changeme \
    --namespace example
  ```
  For unsecure local registry
  ```
  # /etc/rancher/k3s/registries.yaml
  mirrors:
    "registry.home.arpa":
      endpoint:
        - "https://registry.home.arpa"
  
  configs:
    "registry.home.arpa":
      tls:
        insecure_skip_verify: true
  ```

## Maintenance
- Certificate retrieval for Sealed Secrets
  ```shell
  kubeseal \
    --controller-namespace sealed-secrets \
    --controller-name sealed-secrets \
    --fetch-cert > sealed-secrets.cert
  ```
- Creating sealed secrets
  ``` shell
  kubeseal \
    -f secret.yaml \
    -w sealed-secret.yaml \
    --cert sealed-secrets.cert
  ```
