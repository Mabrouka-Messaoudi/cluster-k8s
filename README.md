# k8s-cluster2 — Cluster Kubernetes 2 (Hybride LAN)

Coexiste avec le cluster 1 existant. IPs et CIDRs isolés.

## Structure

```
k8s-cluster2/
├── windows-host/
│   └── Vagrantfile          master (111) + worker1 (112) via VirtualBox
├── linux-host/
│   ├── Vagrantfile          worker2 (113) + worker3 (114) via KVM
│   ├── inventory.ini
│   ├── ansible.cfg
│   ├── site.yml             déploiement complet
│   ├── reset.yml            reset cluster 2 uniquement
│   ├── group_vars/all.yml   variables (CIDRs, IPs)
│   └── roles/
│       ├── common/          préparation système
│       ├── master/          init Control Plane
│       └── worker/          join workers
└── docs/
    └── GUIDE.md             guide complet avec toutes les commandes
```

## Ordre de démarrage

```
1. Linux  : créer br0 si absent   →  docs/GUIDE.md étape 0
2. Windows: vagrant up            →  windows-host/
3. Linux  : vagrant up            →  linux-host/
4. Linux  : ansible-playbook site.yml
```

Voir **docs/GUIDE.md** pour toutes les commandes détaillées.
