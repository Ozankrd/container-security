# Rapport d’Activité – Génération de Clé GPG et Création de Projet GitLab

##Partie 1:

## 1. Création d’un projet GitLab

Dans le cadre de ce TP, nous avons commencé par créer un nouveau projet sur la plateforme GitLab afin de centraliser les travaux liés à l’authentification et à la sécurité des contributions.

### Étapes réalisées

- Connexion au compte GitLab personnel
- Création d’un projet intitulé avec une visibilité **privée**


## 2. Génération d’une paire de clés GPG

Afin de permettre la signature cryptographique des contributions, une paire de clés GPG a été générée localement à l’aide de l’utilitaire GNU Privacy Guard (`gpg`).

gpg --full-generate-key

Paramètres choisis
Type de clé : RSA and RSA (option 1)

Taille : 4096 bits

Durée de validité : 1 an

Identité : ozankrd 

Passphrase : complexe et confidentielle

Résultat
Une clé privée a été générée avec succès et protégée par une passphrase. Elle sera utilisée pour signer les commits.

##3. Récupération de l’ID de la clé
La commande suivante a été exécutée pour identifier l’ID de la clé générée :

gpg --list-secret-keys --keyid-format=long
![1](https://github.com/user-attachments/assets/9f628337-138a-47d9-a8d9-62941a713390)
