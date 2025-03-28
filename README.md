# TP Kubernetes – Déploiement d’un Cluster avec Kind

## 📌 Objectif
Déployer un cluster Kubernetes local en utilisant **Kind (Kubernetes IN Docker)** comportant :
- 2 nœuds **master**
- 2 nœuds **worker**
Puis vérifier l’état du cluster, la liste des namespaces, et la version de Kubernetes utilisée.

---

## 🧱 Étapes réalisées

### 1. Installation de Kind
Kind a été installé via la commande suivante :

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```
![ok](https://github.com/Ozankrd/container-security/blob/s3/1.png)

2. Création du cluster avec 2 masters et 2 workers
Nous avons utilisé un fichier de configuration YAML nommé kind-config.yaml pour définir la structure du cluster.

Contenu du fichier kind-config.yaml :
yaml
Copier
Modifier
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: control-plane
  - role: worker
  - role: worker
Création du cluster :
bash
Copier
Modifier
kind create cluster --config kind-config.yaml --name mon-cluster
3. Vérification de l’état du cluster
Pour vérifier que les nœuds du cluster sont bien créés et actifs :

bash
Copier
Modifier
kubectl get nodes
Résultat attendu :
4 nœuds : 2 en control-plane, 2 en worker

Tous doivent apparaître avec le statut Ready

4. Affichage des namespaces
Pour lister les namespaces disponibles sur le cluster Kubernetes :

bash
Copier
Modifier
kubectl get namespaces
Ou avec l’alias plus court :

bash
Copier
Modifier
kubectl get ns
Résultat attendu :
text
Copier
Modifier
NAME              STATUS   AGE
default           Active   XXm
kube-node-lease   Active   XXm
kube-public       Active   XXm
kube-system       Active   XXm
5. Version de Kubernetes déployée
La version du client et du serveur Kubernetes est obtenue via la commande :

bash
Copier
Modifier
kubectl version --short
Exemple de sortie :
text
Copier
Modifier
Client Version: v1.29.0
Server Version: v1.29.0
