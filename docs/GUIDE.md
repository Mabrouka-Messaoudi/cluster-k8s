# ================================================================
# GUIDE COMPLET — Cluster Kubernetes 2
# Hybrid VirtualBox (Windows) + KVM (Linux) sur LAN 192.168.100.0/24
# Ressources réduites — coexiste avec le cluster 1
# ================================================================

## Récapitulatif des IPs

| Machine         | IP              | Hôte    | RAM    | CPU |
|-----------------|-----------------|---------|--------|-----|
| k8s2-master     | 192.168.100.111 | Windows | 2048MB | 2   |
| k8s2-worker1    | 192.168.100.112 | Windows | 1536MB | 1   |
| k8s2-worker2    | 192.168.100.113 | Linux   | 1024MB | 1   |
| k8s2-worker3    | 192.168.100.114 | Linux   | 1024MB | 1   |

Pod CIDR     : 10.245.0.0/16   ← différent du cluster 1 (10.244.0.0/16)
Service CIDR : 10.97.0.0/12    ← différent du cluster 1 (10.96.0.0/12)

---

## ÉTAPE 0 — Créer le bridge br0 sur le PC Linux (UNE SEULE FOIS)

C'est l'étape la plus importante. Le bridge permet aux VMs KVM
d'être visibles sur ton LAN physique avec leurs propres IPs.

### ⚠️ ATTENTION
Fais ces commandes en CONSOLE LOCALE ou via un deuxième terminal de secours.
Si tu es en SSH et que tu perds la connexion, reconnecte-toi via l'IP physique
de ta machine Linux (192.168.100.10).

### 0.1 — Identifier ton interface réseau physique Linux

```bash
ip link show
```

Tu verras quelque chose comme :
```
1: lo: <LOOPBACK> ...
2: enp3s0: <BROADCAST,MULTICAST,UP> ...     ← c'est elle (exemple)
3: virbr0: ...                               ← KVM virtuel, pas ça
```

Note le nom de ton interface physique. Dans ce guide on utilisera `enp3s0`
comme exemple — remplace par la tienne dans chaque commande.

### 0.2 — Vérifier si br0 existe déjà

```bash
ip link show br0
```

Si tu vois `br0`, saute directement à l'ÉTAPE 1.
Si tu vois `Device "br0" does not exist` → continue.

### 0.3 — Installer les outils nécessaires

```bash
sudo apt update
sudo apt install bridge-utils -y
```

### 0.4 — Trouver la config Netplan existante

```bash
ls /etc/netplan/
cat /etc/netplan/*.yaml
```

Tu verras un fichier comme `00-installer-config.yaml` ou `01-netcfg.yaml`
avec la config de ton interface physique. Note le nom du fichier.

Exemple de ce que tu pourrais voir :
```yaml
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: true
```

### 0.5 — Créer la config du bridge

Remplace `enp3s0` par ton interface réelle.

```bash
# Désactiver la config existante de l'interface physique
# (renommer pour la mettre de côté, pas supprimer)
sudo mv /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.bak

# Créer la nouvelle config avec le bridge
sudo tee /etc/netplan/01-bridge.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp3s0:          # <-- remplace par ton interface physique
      dhcp4: false
  bridges:
    br0:
      interfaces:
        - enp3s0     # <-- même interface ici
      dhcp4: true
      parameters:
        stp: false
        forward-delay: 0
EOF
```

### 0.6 — Appliquer la config

```bash
sudo netplan apply
```

### 0.7 — Vérifier

```bash
ip addr show br0
```

Tu dois voir br0 avec l'IP de ta machine Linux (192.168.100.10) :
```
4: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.100.10/24 brd 192.168.100.255 scope global br0
```

```bash
# Test de connectivité depuis Linux
ping -c 2 192.168.100.6    # ping vers le PC Windows
ping -c 2 8.8.8.8          # ping internet
```

Si les deux fonctionnent, le bridge est bon.

### 0.8 — Si tu perds la connexion SSH

Reconnecte-toi avec l'IP du bridge :
```bash
ssh user@192.168.100.10
```
br0 hérite de l'IP de enp3s0, donc la connexion revient.

---

## ÉTAPE 1 — Préparer le PC Windows

### 1.1 — Trouver le nom de ta carte réseau VirtualBox

Ouvre PowerShell en tant qu'administrateur :

```powershell
VBoxManage list bridgedifs | findstr "Name:"
```

Tu verras quelque chose comme :
```
Name:            Intel(R) Ethernet Connection (2) I219-V
Name:            Realtek PCIe GbE Family Controller
```

Copie le nom exact de ta carte réseau physique (pas le Wi-Fi si possible).

### 1.2 — Modifier le Vagrantfile Windows

Ouvre `windows-host/Vagrantfile` et modifie la ligne :
```ruby
BRIDGE_ADAPTER = "Realtek PCIe GbE Family Controller"  # <-- mets le tien ici
```

### 1.3 — Ouvrir les ports Kubernetes dans le Firewall Windows

Dans PowerShell en tant qu'administrateur :

```powershell
New-NetFirewallRule -DisplayName "K8s-API"     -Direction Inbound -Protocol TCP -LocalPort 6443  -Action Allow
New-NetFirewallRule -DisplayName "K8s-Kubelet" -Direction Inbound -Protocol TCP -LocalPort 10250 -Action Allow
New-NetFirewallRule -DisplayName "K8s-etcd"    -Direction Inbound -Protocol TCP -LocalPort 2379-2380 -Action Allow
New-NetFirewallRule -DisplayName "K8s-NodePort"-Direction Inbound -Protocol TCP -LocalPort 30000-32767 -Action Allow
```

### 1.4 — Démarrer les VMs Windows

```powershell
cd windows-host
vagrant up
```

Vagrant va te demander quelle interface bridger — entre le numéro
correspondant à ta carte LAN physique (pas le Wi-Fi).

### 1.5 — Vérifier les VMs Windows

```powershell
vagrant status
# Doit afficher k8s2-master et k8s2-worker1 en "running"

vagrant ssh k8s2-master
```

Dans la VM :
```bash
ip addr show
# eth1 doit avoir 192.168.100.111

ping 192.168.100.10    # ping Linux host
ping 8.8.8.8           # ping internet
exit
```

---

## ÉTAPE 2 — Démarrer les VMs KVM sur Linux

### 2.1 — Vérifier vagrant-libvirt

```bash
vagrant plugin list | grep libvirt
```

Si absent :
```bash
# Installer les dépendances
sudo apt install ruby-libvirt libvirt-dev -y

# Installer le plugin
vagrant plugin install vagrant-libvirt
```

### 2.2 — Démarrer les VMs Linux

```bash
cd linux-host
vagrant up --provider=libvirt
```

Si tu vois une erreur de pool de stockage :
```bash
sudo virsh pool-list --all
sudo virsh pool-start default      # si le pool existe mais est inactif
# ou
sudo virsh pool-define-as default dir - - - - /var/lib/libvirt/images
sudo virsh pool-build default
sudo virsh pool-start default
sudo virsh pool-autostart default
```

### 2.3 — Vérifier les VMs Linux

```bash
vagrant status
# k8s2-worker2 et k8s2-worker3 doivent être "running"

vagrant ssh k8s2-worker2
```

Dans la VM :
```bash
ip addr show
# La 2ème interface doit avoir 192.168.100.113

ping 192.168.100.111   # ping master Windows
ping 192.168.100.10    # ping Linux host
exit
```

---

## ÉTAPE 3 — Test connectivité complète AVANT Ansible

Depuis le PC Linux, teste TOUTES les IPs :

```bash
for ip in 192.168.100.111 192.168.100.112 192.168.100.113 192.168.100.114; do
  echo -n "Ping $ip : "
  ping -c 1 -W 2 $ip &>/dev/null && echo "OK" || echo "ECHEC"
done
```

Tous doivent répondre OK. Si un échoue → debug réseau avant de continuer.

Tester aussi le SSH :
```bash
for ip in 192.168.100.111 192.168.100.112 192.168.100.113 192.168.100.114; do
  echo -n "SSH $ip : "
  nc -zw 3 $ip 22 && echo "OK" || echo "ECHEC"
done
```

---

## ÉTAPE 4 — Vérifier le nom de l'interface LAN dans les VMs

C'est critique pour Kubernetes. L'interface LAN peut s'appeler différemment.

```bash
# Depuis le dossier linux-host/
vagrant ssh k8s2-master -- "ip -o link show | awk -F': ' '{print \$2}'"
vagrant ssh k8s2-worker2 -- "ip -o link show | awk -F': ' '{print \$2}'"
```

- VirtualBox → souvent `eth1`
- KVM/libvirt → souvent `enp7s0` ou `ens7` ou `eth1`

Si c'est différent de `eth1`, modifie `linux-host/group_vars/all.yml` :
```yaml
lan_interface: "enp7s0"   # mets le nom réel ici
```

---

## ÉTAPE 5 — Lancer Ansible

Depuis le dossier `linux-host/` :

### 5.1 — Tester la connectivité Ansible

```bash
cd linux-host
ansible cluster2 -i inventory.ini -m ping
```

Réponse attendue pour chaque nœud :
```
k8s2-master | SUCCESS => {"ping": "pong"}
```

Si un nœud échoue, teste manuellement :
```bash
ssh -i ~/.vagrant.d/insecure_private_key \
    -o StrictHostKeyChecking=no \
    vagrant@192.168.100.111 "hostname"
```

### 5.2 — Lancer le déploiement complet

```bash
ansible-playbook -i inventory.ini site.yml
```

Durée estimée : **15 à 25 minutes** (dépend de la connexion internet).

Si une tâche échoue, tu peux relancer — Ansible est idempotent :
```bash
ansible-playbook -i inventory.ini site.yml
# Ou depuis une tâche précise :
ansible-playbook -i inventory.ini site.yml --start-at-task="Nom de la tâche"
```

---

## ÉTAPE 6 — Vérifier le cluster 2

```bash
# Se connecter au master
ssh -i ~/.vagrant.d/insecure_private_key vagrant@192.168.100.111

# Sur le master :
kubectl get nodes -o wide
```

Résultat attendu (attendre 2-3 minutes après le join) :
```
NAME           STATUS   ROLES           AGE   VERSION   INTERNAL-IP
k8s2-master    Ready    control-plane   5m    v1.29.x   192.168.100.111
k8s2-worker1   Ready    <none>          3m    v1.29.x   192.168.100.112
k8s2-worker2   Ready    <none>          3m    v1.29.x   192.168.100.113
k8s2-worker3   Ready    <none>          3m    v1.29.x   192.168.100.114
```

```bash
# Vérifier les pods système
kubectl get pods -n kube-system

# Tester un pod nginx
kubectl run nginx-test --image=nginx --restart=Never
kubectl get pod nginx-test -o wide
kubectl delete pod nginx-test
```

---

## ÉTAPE 7 — Travailler avec les deux clusters

### Option A — Switcher avec KUBECONFIG

```bash
# Cluster 1 (existant)
export KUBECONFIG=/chemin/vers/cluster1/admin.conf
kubectl get nodes

# Cluster 2
export KUBECONFIG=/home/vagrant/.kube/config   # sur le master2
# ou copier le fichier :
scp -i ~/.vagrant.d/insecure_private_key \
    vagrant@192.168.100.111:~/.kube/config \
    ~/.kube/config-cluster2

export KUBECONFIG=~/.kube/config-cluster2
kubectl get nodes
```

### Option B — Fusionner les contextes (avancé)

```bash
# Récupérer le kubeconfig cluster2
scp -i ~/.vagrant.d/insecure_private_key \
    vagrant@192.168.100.111:~/.kube/config \
    /tmp/config-cluster2

# Fusionner
KUBECONFIG=~/.kube/config:/tmp/config-cluster2 \
    kubectl config view --flatten > ~/.kube/config-merged

cp ~/.kube/config-merged ~/.kube/config

# Lister les contextes disponibles
kubectl config get-contexts

# Switcher
kubectl config use-context kubernetes-admin@kubernetes   # cluster 1
kubectl config use-context kubernetes-admin@kubernetes   # adapter le nom
```

---

## ÉTAPE 8 — Détruire le cluster 2 (cluster 1 intact)

### Option A : Reset Kubernetes uniquement (VMs conservées)

```bash
cd linux-host
ansible-playbook -i inventory.ini reset.yml
```

### Option B : Détruire toutes les VMs

```bash
# Linux
cd linux-host && vagrant destroy -f

# Windows (PowerShell)
cd windows-host && vagrant destroy -f
```

---

## DÉPANNAGE

### Nœuds en NotReady après le join

```bash
# Sur le master
kubectl describe node k8s2-worker1

# Vérifier les logs kubelet sur le worker
sudo journalctl -u kubelet -n 50

# Vérifier que l'IP LAN est bien utilisée
kubectl get node k8s2-worker1 -o wide
# La colonne INTERNAL-IP doit afficher 192.168.100.112, pas 10.x.x.x
```

Si l'IP est mauvaise :
```bash
# Sur le worker
cat /etc/default/kubelet
# Doit avoir : KUBELET_EXTRA_ARGS=--node-ip=192.168.100.XXX
sudo systemctl restart kubelet
```

### Pods ne communiquent pas entre hôtes différents

```bash
# Vérifier que Flannel utilise la bonne interface
kubectl get daemonset kube-flannel-ds -n kube-flannel -o yaml | grep iface

# Logs Flannel
kubectl logs -n kube-flannel -l app=flannel

# Tester la MTU
ip link show eth1    # MTU doit être <= 1450 pour Flannel VXLAN
```

### Vagrant libvirt : erreur de réseau

```bash
# Vérifier que br0 est actif
ip link show br0

# Vérifier que libvirt peut utiliser br0
sudo virsh net-list --all
sudo brctl show
```

### Vérifier NTP sur tous les nœuds

```bash
ansible cluster2 -i inventory.ini -m shell \
  -a "timedatectl status | grep 'System clock'"
```

Tous doivent afficher `synchronized: yes`.
