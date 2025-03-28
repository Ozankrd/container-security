# Rapport d’Activité – Génération de Clé GPG et Création de Projet GitLab

## Partie 1:

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

## 3. Récupération de l’ID de la clé
La commande suivante a été exécutée pour identifier l’ID de la clé générée :

gpg --list-secret-keys --keyid-format=long
![1](https://github.com/user-attachments/assets/9f628337-138a-47d9-a8d9-62941a713390)

## 4. Exportation de la clé privée pour Cosign

Afin de rendre la clé GPG utilisable par d'autres outils de signature comme Cosign, la clé privée générée précédemment a été exportée sous format ASCII.

gpg --export-secret-keys --armor 3635ECA25C32A239 > private-gpg.key

![1 2](https://github.com/user-attachments/assets/90a2810d-00a7-498c-a75a-8adc3172594d)


## 5. Signature d’une image Docker et publication dans le registre GitLab

### Étape 1 : Construction de l’image Docker

Un fichier `Dockerfile` simple a été rédigé avec le contenu suivant :


FROM alpine
CMD ["echo", "Image signée avec Cosign"]


## Étape 2 : Connexion au registre GitLab
Une authentification a été réalisée via la commande suivante :

docker login registry.gitlab.com
Le nom d’utilisateur GitLab et un Personal Access Token avec les scopes read_registry et write_registry ont permis une connexion réussie.

## Étape 3 : Push de l’image vers GitLab
Une fois connecté, l’image a été poussée vers le Container Registry de GitLab :

docker push registry.gitlab.com/security-container/security-container/image:signee
Le digest SHA-256 a confirmé que l’image était bien stockée dans le registre distant.

![1 3](https://github.com/user-attachments/assets/702b984d-1e5e-4aa6-9b89-5302fbb695f1)

## Étape 4 : Installation de Cosign
L’outil Cosign a été téléchargé et rendu exécutable avec les commandes suivantes :


curl -sSfL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o cosign
chmod +x cosign
La version a été vérifiée via :

./cosign version

## Étape 5 : Tentative de signature avec clé GPG (échec)
Une tentative a été faite pour signer l’image à l’aide de la clé GPG précédemment exportée :


./cosign sign --key gpg://private-gpg.key registry.gitlab.com/security-container/security-container/image:signee
Cependant, cette méthode a échoué avec l’erreur suivante :


Error: ... loading URL: unrecognized scheme: gpg://
Cela s’explique par l’évolution de Cosign : le schéma gpg:// n’est plus pris en charge dans les versions récentes.

## Étape 6 : Génération d’une clé Cosign native
Pour pallier cette limitation, une nouvelle paire de clés a été générée spécifiquement pour Cosign :

./cosign generate-key-pair
Deux fichiers ont été créés :

cosign.key : clé privée (à conserver en sécurité)

cosign.pub : clé publique (peut être utilisée pour la vérification)

Étape 7 : Signature de l’image avec la clé Cosign
La signature de l’image a été réalisée avec succès à l’aide de la clé cosign.key :

./cosign sign --key cosign.key registry.gitlab.com/security-container/security-container/image:signee
Un message de confirmation a validé l’opération de signature.

Étape 8 : Vérification de la signature
Pour garantir l’intégrité et l’authenticité de l’image, une vérification a été effectuée avec la clé publique :

./cosign verify --key cosign.pub registry.gitlab.com/security-container/security-container/image:signee
La sortie de cette commande a confirmé que l’image signée correspond bien à la signature générée précédemment.

![1 4](https://github.com/user-attachments/assets/91da99b4-d430-4b71-bf5a-b732bc5293bf)

## 7. Modification de l’image signée et création d’une version altérée


### Étape 1 : Modification via un conteneur temporaire

Un conteneur a été lancé à partir de l’image signée :

docker run -it registry.gitlab.com/security-container/security-container/image:signee sh

À l’intérieur du conteneur, un fichier fictif a été créé pour simuler une altération :

touch /tampered
Le conteneur a ensuite été quitté.

## Étape 2 : Commit de la version altérée
Le conteneur modifié a été transformé en nouvelle image :

docker commit $(docker ps -lq) registry.gitlab.com/security-container/security-container/image:alteree

## Étape 3 : Push de l’image altérée
La nouvelle image a été poussée vers le registre GitLab :

docker push registry.gitlab.com/security-container/security-container/image:alteree

![1 5](https://github.com/user-attachments/assets/54971109-0f14-4e6f-8030-7b40a44c15a2)


## 8. Vérification de l’authenticité des images avec Cosign

### Objectif

L’objectif de cette étape est de vérifier la signature cryptographique des images Docker à l’aide de **Cosign**, en comparant une image signée avec une version modifiée.

---

### Vérification de l’image signée

La commande suivante a été exécutée :

./cosign verify --key cosign.pub registry.gitlab.com/security-container/security-container/image:signee

Vérification de l’image altérée
Une tentative de vérification a été effectuée sur l’image modifiée :

./cosign verify --key cosign.pub registry.gitlab.com/security-container/security-container/image:alteree

![1 6](https://github.com/user-attachments/assets/0ffa3e79-1310-4016-8ab9-34ae709e8fa0)

