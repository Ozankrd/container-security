
# Rapport d’Activité – Génération de Clé GPG, Signature et Publication d’Images Docker avec Cosign

## Partie 1 : Création du projet et génération de la clé GPG

### 1. Création d’un projet GitLab

Dans le cadre de ce TP, un nouveau projet a été créé sur la plateforme GitLab afin de centraliser les travaux liés à la signature d’images Docker.

#### Étapes réalisées :
- Connexion au compte GitLab personnel
- Création d’un projet intitulé `security-container` avec une visibilité **privée**

---

### 2. Génération d’une paire de clés GPG

Afin de permettre la signature cryptographique des contributions, une paire de clés GPG a été générée localement via la commande suivante :

```bash
gpg --full-generate-key
```

#### Paramètres choisis :
- Type de clé : RSA and RSA (option 1)
- Taille : 4096 bits
- Durée de validité : 1 an
- Identité : `ozankrd <ozan26kordu@gmail.com>`
- Passphrase : complexe et confidentielle

La clé privée a été générée avec succès et protégée par mot de passe.

---

### 3. Récupération de l’ID de la clé

Pour récupérer l’identifiant de la clé, la commande suivante a été utilisée :

```bash
gpg --list-secret-keys --keyid-format=long
```

![1](https://github.com/user-attachments/assets/9f628337-138a-47d9-a8d9-62941a713390)

---

### 4. Exportation de la clé privée pour Cosign

L’export a été réalisé au format ASCII pour permettre son utilisation par des outils de signature :

```bash
gpg --export-secret-keys --armor 3635ECA25C32A239 > private-gpg.key
```

![1 2](https://github.com/user-attachments/assets/90a2810d-00a7-498c-a75a-8adc3172594d)

---

## Partie 2 : Signature d’une image Docker et publication dans le registre GitLab

### Étape 1 : Construction de l’image Docker

Un fichier `Dockerfile` a été rédigé avec le contenu suivant :

```dockerfile
FROM alpine
CMD ["echo", "Image signée avec Cosign"]
```

L’image a été construite avec :

```bash
docker build -t registry.gitlab.com/security-container/security-container/image:signee .
```

---

### Étape 2 : Connexion au registre GitLab

```bash
docker login registry.gitlab.com
```

Connexion réussie à l’aide du nom d’utilisateur GitLab et d’un **Personal Access Token** avec les scopes `read_registry` et `write_registry`.

---

### Étape 3 : Push de l’image vers GitLab

```bash
docker push registry.gitlab.com/security-container/security-container/image:signee
```

L’image a été poussée avec succès dans le registre GitLab.

![1 3](https://github.com/user-attachments/assets/702b984d-1e5e-4aa6-9b89-5302fbb695f1)

---

### Étape 4 : Installation de Cosign

```bash
curl -sSfL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o cosign
chmod +x cosign
./cosign version
```

---

### Étape 5 : Tentative de signature avec clé GPG (échec)

```bash
./cosign sign --key gpg://private-gpg.key registry.gitlab.com/security-container/security-container/image:signee
```

Erreur : `unrecognized scheme: gpg://` → GPG non pris en charge dans cette version de Cosign.

---

### Étape 6 : Génération d’une clé Cosign native

```bash
./cosign generate-key-pair
```

Deux fichiers ont été créés : `cosign.key` (privée), `cosign.pub` (publique).

---

### Étape 7 : Signature de l’image avec la clé Cosign

```bash
./cosign sign --key cosign.key registry.gitlab.com/security-container/security-container/image:signee
```

---

### Étape 8 : Vérification de la signature

```bash
./cosign verify --key cosign.pub registry.gitlab.com/security-container/security-container/image:signee
```

![1 4](https://github.com/user-attachments/assets/91da99b4-d430-4b71-bf5a-b732bc5293bf)

---

## Partie 3 : Modification et détection d’altération

### Étape 1 : Création d’une version modifiée

```bash
docker run -it registry.gitlab.com/security-container/security-container/image:signee sh
touch /tampered
exit
```

### Étape 2 : Commit et push de la version modifiée

```bash
docker commit $(docker ps -lq) registry.gitlab.com/security-container/security-container/image:alteree
docker push registry.gitlab.com/security-container/security-container/image:alteree
```

![1 5](https://github.com/user-attachments/assets/54971109-0f14-4e6f-8030-7b40a44c15a2)

---

## Partie 4 : Vérification de l’authenticité des images

### Image originale (succès)

```bash
./cosign verify --key cosign.pub registry.gitlab.com/security-container/security-container/image:signee
```

### Image altérée (échec)

```bash
./cosign verify --key cosign.pub registry.gitlab.com/security-container/security-container/image:alteree
```

![1 6](https://github.com/user-attachments/assets/0ffa3e79-1310-4016-8ab9-34ae709e8fa0)
