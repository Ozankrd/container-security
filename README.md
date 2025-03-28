# TP Kubernetes – Déploiement d’un Cluster avec Kind

###  Partie 1
Kind a été installé via la commande suivante :

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```
![ok](https://github.com/Ozankrd/container-security/blob/s3/1.png)

Création du cluster avec 2 masters et 2 workers
Nous avons utilisé un fichier de configuration YAML nommé kind-config.yaml pour définir la structure du cluster.

Vérification de l’état du cluster
Pour vérifier que les nœuds du cluster sont bien créés et actifs :

![ok](https://github.com/Ozankrd/container-security/blob/s3/1.1.png)


4 nœuds : 2 en control-plane, 2 en worker

Tous doivent apparaître avec le statut Ready

Pour lister les namespaces disponibles sur le cluster Kubernetes :

![ok](https://github.com/Ozankrd/container-security/blob/s3/1.3.png)

# TP – Déploiement d’un Cluster Kubernetes avec Kind

## Partie 2 : Expérimentation des RBAC (Role-Based Access Control)

---

## 1. Création d’un namespace dédié

Nous avons commencé par créer un namespace nommé `test-rbac`, destiné à l’expérimentation des règles RBAC.




