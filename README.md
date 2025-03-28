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

![ok](https://github.com/Ozankrd/container-security/blob/s3/1.2.png)


4 nœuds : 2 en control-plane, 2 en worker

Tous doivent apparaître avec le statut Ready

Pour lister les namespaces disponibles sur le cluster Kubernetes :

![ok](https://github.com/Ozankrd/container-security/blob/s3/1.3.png)


## Partie 2 : Expérimentation des RBAC (Role-Based Access Control)

---

## 1. Création d’un namespace dédié

Nous avons commencé par créer un namespace nommé `test-rbac`, destiné à l’expérimentation des règles RBAC.

![1 4](https://github.com/user-attachments/assets/bd65efc0-020d-44e8-b6e4-f749f487ae37)*

## 2. Déploiement d’un pod dans le namespace test-rbac
Le fichier mon-pod.yaml contient la définition suivante :

apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: test-rbac
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
Application de la configuration :
![1 5](https://github.com/user-attachments/assets/84fa326f-c5bb-4cda-9494-e6df79e9949a)

##
3. Consultation des logs du pod
Pour afficher les logs du pod nginx dans le namespace test-rbac, la commande suivante a été utilisée :

kubectl logs nginx -n test-rbac

![1 6](https://github.com/user-attachments/assets/22f2c990-316e-4746-b024-59e097ef0667)


## 4. Création d’un rôle RBAC : pod-reader
Le rôle a été défini dans le fichier role-pod-reader.yaml :

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test-rbac
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
Application du rôle :

kubectl apply -f role-pod-reader.yaml
## 5. Vérification du rôle
Nous avons affiché la définition du rôle avec :

kubectl get role pod-reader -n test-rbac -o yaml

![1 7](https://github.com/user-attachments/assets/19a92db3-99c8-4560-84a3-657a1c130a39)


## 5. Liaison du rôle à l’utilisateur titi
Nous avons défini un RoleBinding dans le fichier rolebinding-pod-reader.yaml :

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: test-rbac
subjects:
- kind: User
  name: titi
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
Application :

kubectl apply -f rolebinding-pod-reader.yaml
## 6. Création de l’utilisateur titi avec certificats
a. Récupération des certificats CA du cluster Kind :

docker cp mon-cluster-control-plane:/etc/kubernetes/pki/ca.crt .
docker cp mon-cluster-control-plane:/etc/kubernetes/pki/ca.key .
b. Génération des clés pour l’utilisateur titi :

openssl genrsa -out titi.key 2048
openssl req -new -key titi.key -out titi.csr -subj "/CN=titi"
openssl x509 -req -in titi.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out titi.crt -days 365
## 7. Ajout de l’utilisateur dans la configuration kubectl

kubectl config set-credentials titi \
  --client-certificate=titi.crt \
  --client-key=titi.key
## 8. Création et utilisation du contexte titi-context
Création d’un contexte propre à l’utilisateur titi :

kubectl config set-context titi-context \
  --cluster=kind-kind \
  --namespace=test-rbac \
  --user=titi
Activation du contexte :


kubectl config use-context titi-context
## 9. Vérification des permissions de l’utilisateur titi
Lecture des pods (autorisé) :

kubectl get pods

## 10. Retour au contexte administrateur

kubectl config use-context kind-kind

![1 8](https://github.com/user-attachments/assets/81144088-1687-491c-909c-fae3c86a9891)
![1 9](https://github.com/user-attachments/assets/bb18a594-0c5f-4e9b-b44f-d022c53e3637)

