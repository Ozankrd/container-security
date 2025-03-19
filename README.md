# Rapport de TP : Sécurité des Conteneurs Docker

## Introduction
Ce TP vise à explorer les bonnes pratiques de sécurité lors de l'utilisation de conteneurs Docker. Nous allons examiner des techniques pour restreindre l'exposition des ports, sécuriser les fichiers sensibles, auditer la configuration des conteneurs et gérer les secrets avec HashiCorp Vault.

## 1. Éviter l’Exposition Involontaire de Ports
L'objectif est de comprendre comment limiter l'exposition des ports d'un conteneur Docker.

### Étapes :
1. Lancer un conteneur **nginx** en exposant uniquement le port 8080 :
   ```bash
   docker run -d -p 8080:80 nginx
   ```
2. Vérifier si le port est bien exposé avec la commande :
   ```bash
   netstat -tulnp | grep 8080
   ```
   ou
   ```bash
   ss -tulnp | grep 8080
   ```
3. Confirmer que le conteneur est accessible uniquement sur **localhost**.

## 2. Restreindre les Permissions d’Accès aux Fichiers Sensibles
On va monter un volume en mode lecture seule pour éviter toute modification accidentelle.

### Étapes :
1. Lancer un conteneur Alpine en montant le fichier `/etc/passwd` en lecture seule :
   ```bash
   docker run -it --rm -v /etc/passwd:/mnt/passwd:ro alpine sh
   ```
2. Tester la lecture du fichier :
   ```bash
   cat /mnt/passwd
   ```
3. Tester l’écriture dans le fichier :
   ```bash
   echo "test" >> /mnt/passwd
   ```
   
   **Résultat attendu :** L’écriture doit être refusée en raison des permissions restreintes.

## 3. Auditer la Configuration d’un Conteneur avec Docker Bench
Docker Bench permet d’analyser les configurations de sécurité de l’hôte et des conteneurs.

### Étapes :
1. Cloner le dépôt GitHub de Docker Bench et exécuter l’audit :
   ```bash
   git clone https://github.com/docker/docker-bench-security.git
   cd docker-bench-security/
   ./docker-bench-security.sh
   ```
2. Noter le score obtenu pour l’hôte Docker.
3. Auditer le conteneur **vulnerables/web-dvwa** :
   ```bash
   docker run --rm -it vulnerables/web-dvwa sh
   ```
4. Identifier les vulnérabilités potentielles et les permissions excessives.

## 4. Stocker et Utiliser des Secrets avec HashiCorp Vault
L’objectif est de comprendre comment gérer des secrets de manière sécurisée.

### Étapes :
1. Lancer un conteneur Vault :
   ```bash
   docker run --cap-add=IPC_LOCK -e 'VAULT_LOCAL_CONFIG={"storage": {"file": {"path": "/vault/file"}}, "listener": [{"tcp": { "address": "0.0.0.0:8200", "tls_disable": true}}], "default_lease_ttl": "168h", "max_lease_ttl": "720h", "ui": true}' -p 8200:8200 vault:1.13.3 server
   ```
2. Accéder à l’interface utilisateur à l’adresse **http://localhost:8200**.
3. Se connecter avec le **root token** fourni lors de l’initialisation.
4. Créer une authentification **user/mot de passe**.
5. Créer une **ACL** permettant d’accéder au chemin **containers/mon-secret**.
6. Associer l’ACL à l’utilisateur créé.
7. Ajouter un secret dans **containers/mon-secret**.
8. Lancer un conteneur Alpine et récupérer le secret via l’API Vault :
   ```bash
   TOKEN=$(curl --request POST --data '{"password":"password123"}' http://host.docker.internal:8200/v1/auth/userpass/login/user1 | jq -r .auth.client_token)
   curl --header "X-Vault-Token: $TOKEN" http://host.docker.internal:8200/v1/kv/data/containers/mon-secret
   ```

## 5. Trouver une Clé API Cachée dans une Image Docker
L’objectif est de simuler une attaque pour récupérer une clé API cachée dans une image Docker mal configurée.

### Étapes :
1. Télécharger l’image suspecte :
   ```bash
   docker pull ety92/demo:v1
   ```
2. Analyser l’historique des couches Docker pour repérer des fuites d’informations :
   ```bash
   docker history --no-trunc ety92/demo:v1
   ```
3. On observe une ligne suspecte dans l’historique contenant :
   ```
   RUN /bin/sh -c apk add curl && curl -H "API-Key: U-never-will-saw-that" -L google.com
   ```
   Cela signifie que la clé API a été enregistrée dans une des couches de l’image.

### Comment le Développeur Aurait Dû Procéder ?
- Ne pas inclure les clés API directement dans le **Dockerfile**.
- Utiliser **les variables d’environnement** ou **un fichier .env** non versionné.
- Nettoyer les couches d’historique après ajout de données sensibles.
- Stocker les secrets dans un **gestionnaire sécurisé** comme HashiCorp Vault.

## Conclusion
Ce TP a permis d’explorer différentes vulnérabilités de sécurité liées aux conteneurs Docker et comment les prévenir. La gestion des ports, des fichiers sensibles, l’audit de sécurité et la protection des secrets sont des éléments clés pour garantir la sécurité d’un environnement Docker en production.

**Recommandations finales :**
- Toujours limiter l’exposition des services.
- Vérifier les permissions des fichiers montés.
- Auditer régulièrement les conteneurs.
- Ne jamais stocker de secrets en dur dans les images Docker.
- Utiliser des solutions dédiées pour la gestion des secrets.

---
**Auteur : [Ton Nom]**  
**Date : [Date du TP]**
