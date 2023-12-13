# Démo kube-hetzner

## Prérequis…

* Avoir un compte Hetzner, créer un projet et sa clef d'api (token) pleins droits

* Créer les snapshots de microos avec packer (dans hetzner)

Avec le script

```bash
tmp_script=$(mktemp) && curl -sSL -o "${tmp_script}" https://raw.githubusercontent.com/kube-hetzner/terraform-hcloud-kube-hetzner/master/scripts/create.sh && chmod +x "${tmp_script}" && "${tmp_script}" && rm "${tmp_script}"
```
Ou à la main…

```bash
mkdir demo
cd demo
curl -sL https://raw.githubusercontent.com/kube-hetzner/terraform-hcloud-kube-hetzner/master/kube.tf.example -o kube.tf
curl -sL https://raw.githubusercontent.com/kube-hetzner/terraform-hcloud-kube-hetzner/master/packer-template/hcloud-microos-snapshots.pkr.hcl -o hcloud-microos-snapshots.pkr.hcl
export HCLOUD_TOKEN="your_hcloud_token"
packer init hcloud-microos-snapshots.pkr.hcl
packer build hcloud-microos-snapshots.pkr.hcl
hcloud context create demo
```

* Générer une clef SSH *sans passphrase* pour se connecter aux machines

```bash
# dans votre .ssh ...
ssh-keygen -t ed25519 -f demo
```

> Terraform en aura besoin pour… terraformer

## Configuration du cluster

On va customiser notre fichier `kube.tf` avant, pour choisir ce qui nous intéresse...

* Les nœuds dans `agent_nodepools` et `control_plane_nodepools` qu'on veut
  * par exemple 3 noeuds control plane et 2 nœud worker/agent
* `network_region = "eu-central"`
* Pour économiser de cacahuète
  * `allow_scheduling_on_control_plane`, mais on evitera en prod...
  * `load_balancer_location` et `load_balancer_type`, a commenter, (on a déjà des ips publiques avec lesquels s'amuser)
  * `disable_hetzner_csi`, a moins qu'on veuille payer pour ça ...
    * `enable_longhorn = true` à la place pour utiliser l'espace libre des noeud !
    * `longhorn_fstype = "xfs"` Pour faire du RWX ! (ext4 par defaut sinon)
* `cni_plugin =` "cilium" ou "calico" ou "flannel" !
* `create_kubeconfig = false`, question d'*opseq*
* Desactiver les trucs par défaut de k3s... si vous voulez
  * `enable_cert_manager = false` 
  * `enable_klipper_metal_lb` je n'ai jamais eu une bonne expérience avec...
  * `k3s_exec_server_args = "--disable traefik"` Pareil, je préfère installer moi-même
* `cluster_name =` nom du cluster, disons "pizza"
* Bonus...
  * `enable_wireguard` Pour avoir un lan encore plus sécurisé
* Et pleins d'autres choses (autoscaling, etc) <https://github.com/kube-hetzner/terraform-hcloud-kube-hetzner/blob/master/docs/terraform.md>


## Déploiement !

```bash
cd demo
source credentials.env
export TF_VAR_hcloud_token
terraform init --upgrade
terraform validate
terraform apply -auto-approve
...
terraform output --raw kubeconfig >>  ~/.kube/config
```

Démo terminé
```bash
terraform destroy -auto-approve
```