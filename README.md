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


## Partie 3 :

## 1. Déploiement de kube-bench via un Job Kubernetes

L’outil kube-bench a été exécuté à l’intérieur du cluster en créant une ressource de type `Job`. Ce choix permet d’exécuter le scan de manière éphémère et isolée, sans perturber les composants du cluster.

Fichier `job.yml` :
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    spec:
      containers:
      - name: kube-bench
        image: aquasec/kube-bench:latest
        command: ["kube-bench", "--benchmark", "cis-1.23"]
        volumeMounts:
        - name: var-lib-etcd
          mountPath: /var/lib/etcd
        - name: etc-systemd
          mountPath: /etc/systemd
        - name: etc-kubernetes
          mountPath: /etc/kubernetes
        - name: usr-bin
          mountPath: /usr/bin
      restartPolicy: Never
      hostPID: true
      volumes:
      - name: var-lib-etcd
        hostPath:
          path: /var/lib/etcd
      - name: etc-systemd
        hostPath:
          path: /etc/systemd
      - name: etc-kubernetes
        hostPath:
          path: /etc/kubernetes
      - name: usr-bin
        hostPath:
          path: /usr/bin
  backoffLimit: 0

Déploiement :

kubectl apply -f job.yml
Le pod s’est exécuté puis est passé en état Completed.
![1 10](https://github.com/user-attachments/assets/a980f7fe-a725-4145-9f24-e41210228ae7)

## 2. Consultation des résultats du scan
Les résultats du benchmark ont été récupérés 

Ce rapport comprend une série de recommandations regroupées par composant : API Server, Scheduler, Controller Manager, etc. Chaque test est annoté par un statut :

[PASS] : conforme

[FAIL] : non conforme

[WARN] : attention requise

[INFO] : information seulement

## 3. Résumé des résultats du benchmark
Dans le contexte d’un cluster Kind exécuté localement, les résultats observés sont généralement les suivants :

De nombreux tests sont passés avec succès, notamment ceux liés aux permissions de fichiers et à la configuration de base du kubelet.

Quelques avertissements ([WARN]) apparaissent pour des fonctionnalités non activées dans un environnement de développement (ex. : audit logging, sécurité des communications TLS).

Des échecs ([FAIL]) sont fréquemment observés sur des paramètres avancés de l’API Server ou sur l’absence d’authentification forte entre composants, ce qui est attendu dans un environnement non productif.

Ces résultats doivent être interprétés avec nuance : un cluster de test (comme Kind) n'est pas conçu pour être totalement conforme aux normes de production CIS.

![1 11](https://github.com/user-attachments/assets/b2c13be7-03fd-497a-8624-52782135ee8c)



## Partie 4:

## 1. Ajout du dépôt Helm Falco

Falco étant distribué via Helm, nous avons commencé par ajouter le dépôt officiel :

helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

## 2. Ajout du dépôt Helm de Falco
Nous avons ajouté le dépôt officiel de Falco puis mis à jour les charts :


helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update


## 3. Création du namespace dédié
Un namespace falco a été créé pour isoler les ressources :


kubectl create ns falco
## 4. Déploiement de Falco avec Falcosidekick UI
Nous avons utilisé Helm pour installer Falco, en activant à la fois Falcosidekick et son interface web :

helm -n falco install falco falcosecurity/falco \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true
Le déploiement a démarré l’ensemble des composants suivants :

falco (détecteurs sur chaque nœud)

falcosidekick (gestionnaire d’alertes)

falcosidekick-ui (interface web)

redis (backend pour l'UI)

## 5. Vérification de l’état des pods
Quelques minutes après le déploiement, tous les pods sont passés à l’état Running :


kubectl get pods -n falco
Extrait observé :

![1 12](https://github.com/user-attachments/assets/e3ca18a6-0979-4bca-bc5f-3120454425ed)


## 6. Accès à l’interface web (Falcosidekick UI)
Pour consulter l’interface graphique, nous avons exposé le service à l’aide d’un port-forward :

kubectl port-forward svc/falco-falcosidekick-ui 2802:2802 -n falco
En accédant à l’URL suivante dans le navigateur :  http://127.0.0.1:2802

Nous avons pu afficher le dashboard en temps réel de FalcoSidekick UI, où s’affichent les événements et alertes détectés dans le cluster (exécution de commandes, accès au shell, manipulations réseau, etc.).

## 7. Fonctionnement de Falco
Falco surveille en continu les activités suspectes dans le cluster. Il s’appuie sur des règles définies pour détecter :

L’ouverture d’un shell interactif dans un conteneur

Des accès non autorisés à des fichiers sensibles

Des processus anormaux exécutés dans des pods

Des connexions réseau suspectes

Les événements détectés sont envoyés vers Falcosidekick, qui les transmet à l’interface UI.

![1 13](https://github.com/user-attachments/assets/f25295f4-1055-487d-8af0-1483e403f2b5)

## Partie 5

## 1. Déploiement d’un pod `front` de type Alpine

Le pod `front` a été défini dans le fichier `mon-pod.yml` :

apiVersion: v1
kind: Pod
metadata:
  labels:
    app: front
  name: front
spec:
  containers:
  - image: alpine
    name: front
    command:
    - /bin/sh
    - -c
    - sleep 1d
Application :

kubectl apply -f mon-pod.yml
## 2. Génération d’une alerte via ouverture de shell
Nous avons ouvert un terminal interactif dans le pod :

kubectl exec -it front -- sh
Résultat observé dans l’interface Falco :
Règle déclenchée : Terminal shell in container

Priorité : Notice

Comportement détecté : lancement d’un shell (sh) dans un conteneur, ce qui est typiquement utilisé lors d’attaques manuelles ou de mouvements latéraux.

## 3. Génération d’une alerte via accès à l’API Kubernetes
Depuis le shell, nous avons installé curl et effectué une requête vers l’API :

apk add curl
curl -k http://10.96.0.1:80
10.96.0.1 est l’adresse par défaut du service API Kubernetes dans un cluster Kind.

Résultats observés dans Falco :
Alerte n°1
Règle : Contact K8S API Server From Container

Priorité : Notice

Explication : détection d'une connexion TCP sortante depuis un conteneur vers le serveur API Kubernetes – action typique d’un conteneur compromis essayant d’explorer le cluster.

Alerte n°2
Règle : Drop and execute new binary in container

Priorité : Critical

Explication : Falco a détecté l’exécution d’un binaire (curl) non présent dans l’image de base (alpine), signalant une tentative d’introduction et d'exécution d’un outil externe, potentiellement malveillant.

## 4. Analyse des alertes dans Falcosidekick UI
Toutes les alertes ont été consultées depuis l’interface web :

http://127.0.0.1:2802

Les informations suivantes étaient disponibles :

Date et heure précise de l’événement

Nom du conteneur (front)

Image utilisée (docker.io/library/alpine)

Commandes exécutées : sh, apk add curl, curl -k ...

UID utilisateur (root)

Niveau de sévérité (Notice, Critical)

Règle associée à l’alerte (ex. execve, connect)

Références MI!
TRE ATT&CK pour classification (ex : T1059, TA0003, T1565)


[1 14](https://github.com/user-attachments/assets/a76ee305-869f-4358-9b2c-84281feed1bd)
