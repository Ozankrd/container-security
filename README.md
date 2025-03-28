
# TP Kubernetes – Déploiement d’un Cluster avec Kind

## Partie 1 : Déploiement du Cluster avec Kind

### 1. Installation de Kind

Kind a été installé via la commande suivante :

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

![ok](https://github.com/Ozankrd/container-security/blob/s3/1.png)

### 2. Création du cluster avec 2 masters et 2 workers

Le cluster a été créé en utilisant un fichier de configuration YAML nommé `kind-config.yaml` pour définir la structure du cluster.

### 3. Vérification de l’état du cluster

Pour vérifier que les nœuds du cluster sont bien créés et actifs :

![ok](https://github.com/Ozankrd/container-security/blob/s3/1.2.png)

Le cluster contient 4 nœuds : 2 en **control-plane** et 2 en **worker**. Tous doivent apparaître avec le statut **Ready**.

### 4. Liste des namespaces

Pour lister les namespaces disponibles sur le cluster Kubernetes :

![ok](https://github.com/Ozankrd/container-security/blob/s3/1.3.png)

---

## Partie 2 : Expérimentation des RBAC (Role-Based Access Control)

### 1. Création d’un namespace dédié

Nous avons créé un namespace nommé `test-rbac` destiné à l’expérimentation des règles RBAC.

![1 4](https://github.com/user-attachments/assets/bd65efc0-020d-44e8-b6e4-f749f487ae37)

### 2. Déploiement d’un pod dans le namespace `test-rbac`

Le fichier `mon-pod.yaml` contient la définition suivante :

```yaml
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
```

Application de la configuration :

![1 5](https://github.com/user-attachments/assets/84fa326f-c5bb-4cda-9494-e6df79e9949a)

### 3. Consultation des logs du pod

Pour afficher les logs du pod `nginx` dans le namespace `test-rbac`, la commande suivante a été utilisée :

```bash
kubectl logs nginx -n test-rbac
```

![1 6](https://github.com/user-attachments/assets/22f2c990-316e-4746-b024-59e097ef0667)

### 4. Création d’un rôle RBAC : `pod-reader`

Le rôle a été défini dans le fichier `role-pod-reader.yaml` :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test-rbac
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

Application du rôle :

```bash
kubectl apply -f role-pod-reader.yaml
```

### 5. Vérification du rôle

Nous avons affiché la définition du rôle avec :

```bash
kubectl get role pod-reader -n test-rbac -o yaml
```

![1 7](https://github.com/user-attachments/assets/19a92db3-99c8-4560-84a3-657a1c130a39)

### 6. Liaison du rôle à l’utilisateur `titi`

Un `RoleBinding` a été défini dans le fichier `rolebinding-pod-reader.yaml` :

```yaml
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
```

Application du `RoleBinding` :

```bash
kubectl apply -f rolebinding-pod-reader.yaml
```

### 7. Création de l’utilisateur `titi` avec certificats

a. Récupération des certificats CA du cluster Kind :

```bash
docker cp mon-cluster-control-plane:/etc/kubernetes/pki/ca.crt .
docker cp mon-cluster-control-plane:/etc/kubernetes/pki/ca.key .
```

b. Génération des clés pour l’utilisateur `titi` :

```bash
openssl genrsa -out titi.key 2048
openssl req -new -key titi.key -out titi.csr -subj "/CN=titi"
openssl x509 -req -in titi.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out titi.crt -days 365
```

### 8. Ajout de l’utilisateur dans la configuration kubectl

```bash
kubectl config set-credentials titi   --client-certificate=titi.crt   --client-key=titi.key
```

### 9. Création et utilisation du contexte `titi-context`

Création d’un contexte propre à l’utilisateur `titi` :

```bash
kubectl config set-context titi-context   --cluster=kind-kind   --namespace=test-rbac   --user=titi
```

Activation du contexte :

```bash
kubectl config use-context titi-context
```

### 10. Vérification des permissions de l’utilisateur `titi`

Lecture des pods (autorisé) :

```bash
kubectl get pods
```

### 11. Retour au contexte administrateur

```bash
kubectl config use-context kind-kind
```

![1 8](https://github.com/user-attachments/assets/81144088-1687-491c-909c-fae3c86a9891)
![1 9](https://github.com/user-attachments/assets/bb18a594-0c5f-4e9b-b44f-d022c53e3637)

---

## Partie 3 : Déploiement de kube-bench via un Job Kubernetes

### 1. Déploiement de kube-bench

L’outil kube-bench a été exécuté à l’intérieur du cluster en créant une ressource de type `Job`. Ce choix permet d’exécuter le scan de manière éphémère et isolée.

Fichier `job.yml` :

```yaml
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
```

Déploiement :

```bash
kubectl apply -f job.yml
```

Le pod s’est exécuté puis est passé en état **Completed**.

![1 10](https://github.com/user-attachments/assets/a980f7fe-a725-4145-9f24-e41210228ae7)

### 2. Consultation des résultats du scan

Les résultats du benchmark ont été récupérés et analysés. Ils contiennent une série de recommandations regroupées par composant, telles que :

- [PASS] : conforme
- [FAIL] : non conforme
- [WARN] : attention requise
- [INFO] : information seulement

---

## Partie 4 : Déploiement de Falco avec Helm

### 1. Ajout du dépôt Helm Falco

Falco est distribué via Helm. Le dépôt officiel a été ajouté via :

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

### 2. Création du namespace dédié

Un namespace `falco` a été créé pour isoler les ressources :

```bash
kubectl create ns falco
```

### 3. Déploiement de Falco avec Falcosidekick UI

Falco a été déployé avec Helm, en activant Falcosidekick et son interface web :

```bash
helm -n falco install falco falcosecurity/falco   --set falcosidekick.enabled=true   --set falcosidekick.webui.enabled=true
```

### 4. Vérification de l’état des pods

Les pods sont passés à l’état **Running** après quelques minutes.

```bash
kubectl get pods -n falco
```

![1 12](https://github.com/user-attachments/assets/e3ca18a6-0979-4bca-bc5f-3120454425ed)

### 5. Accès à l’interface web

Le service a été exposé avec un port-forward pour accéder à l’interface web de Falcosidekick :

```bash
kubectl port-forward svc/falco-falcosidekick-ui 2802:2802 -n falco
```

Accès à l’UI : [http://127.0.0.1:2802](http://127.0.0.1:2802)

---

## Partie 5 : Déploiement d’un pod `front` de type Alpine

### 1. Déploiement du pod `front`

Le pod `front` a été créé avec le fichier `mon-pod.yml` :

```yaml
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
```

### 2. Génération d’une alerte via ouverture de shell

Une alerte a été générée en ouvrant un terminal interactif dans le pod :

```bash
kubectl exec -it front -- sh
```

### 3. Alerte via accès à l’API Kubernetes

Une requête vers l’API Kubernetes a été effectuée depuis le shell :

```bash
apk add curl
curl -k http://10.96.0.1:80
```

### 4. Analyse des alertes dans Falcosidekick UI

Les alertes ont été consultées dans l’interface web de Falcosidekick UI, avec les informations suivantes :

- Date et heure de l’événement
- Conteneur concerné
- Règle associée (ex. `execve`, `connect`)
- Niveau de sévérité (Notice, Critical)

![1 14](https://github.com/user-attachments/assets/59d4f64c-8d66-4441-a770-fc1a463d6bfc)
