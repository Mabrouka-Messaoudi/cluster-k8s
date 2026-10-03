# cluster-k8s – Cluster Kubernetes hybride (VirtualBox + KVM)

Automatisation (Vagrant + Ansible) d'un cluster Kubernetes de 4 nœuds réparti sur deux machines
physiques du même LAN. C'est la **base d'infrastructure** de mon PFE : la plateforme de microservices
NexShop et son monitoring sont déployés par-dessus, voir le dépôt
[`microservices-java-k8s-monitoring`](https://github.com/Mabrouka-Messaoudi/microservices-java-k8s-monitoring).

Ce cluster (« cluster 2 ») coexiste avec un premier cluster : ses IP et ses CIDR sont volontairement différents.

## Architecture

| Nœud | IP | Hôte | Hyperviseur | RAM | CPU | Rôle |
|------|----|------|-------------|-----|-----|------|
| `k8s2-master` | 192.168.100.111 | PC Windows | VirtualBox | 2 Go | 2 | control-plane |
| `k8s2-worker1` | 192.168.100.112 | PC Windows | VirtualBox | 1,5 Go | 1 | worker |
| `k8s2-worker2` | 192.168.100.113 | PC Linux | KVM / libvirt | 1 Go | 1 | worker |
| `k8s2-worker3` | 192.168.100.114 | PC Linux | KVM / libvirt | 1 Go | 1 | worker |

- Réseau LAN : `192.168.100.0/24` (VMs en mode bridge, donc visibles sur le réseau physique)
- Pods : `10.245.0.0/16` · Services : `10.97.0.0/12` (différents du cluster 1)
- Kubernetes 1.29 (kubeadm), runtime containerd, CNI Flannel (VXLAN)

## Structure

```
windows-host/Vagrantfile   VMs master + worker1 (VirtualBox)
linux-host/
├── Vagrantfile            VMs worker2 + worker3 (KVM / libvirt)
├── ansible.cfg
├── inventory.ini          nœuds et clés SSH
├── group_vars/all.yml     variables : IP du master, CIDR, version, CNI
├── host_vars/             interface réseau par nœud
├── site.yml               déploiement complet
├── reset.yml              réinitialise le cluster 2 uniquement
└── roles/
    ├── common/            préparation système, containerd, kubeadm/kubelet/kubectl
    ├── master/            kubeadm init, Flannel, génération de la commande join
    └── worker/            kubeadm join
docs/GUIDE.md              guide pas à pas avec toutes les commandes
```

## Prérequis

- **PC Linux** : Vagrant avec le plugin `vagrant-libvirt`, KVM/libvirt, Ansible, un bridge `br0` (guide, étape 0)
- **PC Windows** : Vagrant et VirtualBox
- Les deux machines sur le même LAN, accès SSH du PC Linux vers le PC Windows (saut SSH utilisé par l'inventaire)

## Démarrage rapide

```
1. Linux   : créer le bridge br0 si absent                 → docs/GUIDE.md, étape 0
2. Windows : cd windows-host && vagrant up
3. Linux   : cd linux-host && vagrant up --provider=libvirt
4. Linux   : copier les clés SSH du master et du worker1    → docs/GUIDE.md, étape 4 bis
5. Linux   : cd linux-host && ansible cluster2 -m ping
6. Linux   : ansible-playbook -i inventory.ini site.yml     (15 à 25 minutes)
```

Vérification sur le master : `kubectl get nodes -o wide` doit afficher les 4 nœuds `Ready`.
Toutes les commandes détaillées, le dépannage et la destruction du cluster sont dans
**[docs/GUIDE.md](docs/GUIDE.md)**.

## Ce que font les playbooks

- **common** (tous les nœuds) : désactive le swap, charge `overlay` et `br_netfilter`, règle sysctl,
  installe containerd (cgroup systemd) puis kubelet, kubeadm et kubectl en version figée, force kubelet
  à utiliser l'IP du LAN (`--node-ip`).
- **master** : `kubeadm init` avec les CIDR du cluster 2, copie du kubeconfig, installation de Flannel
  configuré sur l'interface LAN, génération de la commande `kubeadm join`.
- **worker** : lit la commande join et rejoint le cluster.

Les playbooks sont idempotents : en cas d'échec, relancer `site.yml`. `reset.yml` remet à zéro le cluster 2
sans toucher au cluster 1.

## Sécurité et limites connues

- `.vagrant/` (qui contient les clés SSH privées générées par Vagrant) n'est **pas versionné**.
  Ne jamais le commiter.
- Le token de join est écrit dans `/tmp/k8s2_join_command.sh` sur la machine qui lance Ansible
  (droits `0600`, validité 24 h par défaut).
- Les VMs sont créées avec `--ignore-preflight-errors=NumCPU,Mem` car les ressources sont volontairement
  réduites : configuration adaptée à un banc d'essai, pas à la production.
- Le manifest Flannel est téléchargé depuis la dernière version publiée (`latest`) : pour une installation
  reproductible, figer la version dans `roles/master/tasks/main.yml`.
- Flannel reçoit une seule option `--iface` pour tout le cluster. Si les interfaces LAN n'ont pas le même nom
  sur tous les nœuds, utiliser `--iface-regex` à la place.
- Les IP des workers KVM dans `inventory.ini` (192.168.121.x) sont attribuées par DHCP et peuvent changer
  après un `vagrant destroy`.

## Nettoyage

```bash
cd linux-host && ansible-playbook -i inventory.ini reset.yml   # reset Kubernetes, VMs conservées
cd linux-host && vagrant destroy -f                            # détruit les VMs Linux
cd windows-host && vagrant destroy -f                          # détruit les VMs Windows
```
