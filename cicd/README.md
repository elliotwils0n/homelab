# cicd

## Configuration
### Cluster
- Install [Docker Engine](https://docs.docker.com/engine/install/) for Forgejo runner
- Install [k3s](https://docs.k3s.io/quick-start)
  ```shell
  curl -sfL https://get.k3s.io | sh -s -
  ```
  optionally, with traefik disabled
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
- to resolve host for homelab without registered domain add records to coredns
  <details>
  <summary>records</summary>

  ```
  192.168.0.200 forgejo.home.arpa
  #192.168.0.200 gitlab.home.arpa
  192.168.0.200 registry.home.arpa
  192.168.0.200 tempo.home.arpa
  192.168.0.200 loki.home.arpa
  192.168.0.200 prometheus.home.arpa
  192.168.0.200 keycloak.home.arpa
  ```
  </details>

  ```shell
  kubectl -n kube-system edit cm coredns
  kubectl -n kube-system rollout restart deploy coredns
  ```

- Move initial cluster manifests to [k3s AddOns directory](https://docs.k3s.io/installation/packaged-components)
  ```
  sudo cp configuration/* /var/lib/rancher/k3s/server/manifests/.
  ```

### Post bootstrap
- Initialize Keycloak `homelab` realm, clients (argocd, forgejo, grafana) and scopes (groups)
- Create argo-bot account in Forgejo
- Login into forgejo as admin and get token from `https://{forgejo url}/admin/actions/runners`
  and update token for `cicd/services/forgejo/runner/secret.yaml`

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

## /etc/hosts records
```
192.168.0.200 cicd.home.arpa
192.168.0.200 forgejo.home.arpa
#192.168.0.200 gitlab.home.arpa
192.168.0.200 argocd.home.arpa
192.168.0.200 grafana.home.arpa
192.168.0.200 keycloak.home.arpa
```
| service    | address                      | description          |
|------------|------------------------------|----------------------|
| forgejo    | https://forgejo.home.arpa    | git repositories     |
| ~~gitlab~~ | ~~https://gitlab.home.arpa~~ | ~~git repositories~~ |
| argocd     | https://argocd.home.arpa     | cd                   |
| grafana    | https://grafana.home.arpa    | monitoring           |
| keycloak   | https://keycloak.home.arpa   | identity management  |
