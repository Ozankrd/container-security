# Activités Pratiques : Sécurité des Conteneurs Docker

## Objectif
Ce projet regroupe différentes expérimentations visant à comprendre les vulnérabilités des conteneurs Docker et à les sécuriser. Il inclut des tests d’exécution avec privilèges, des simulations d’évasion, la création d’une image sécurisée, la restriction réseau, ainsi que l’analyse de vulnérabilités avec **Trivy** et **Grype**.

---

## 1. Tester un Conteneur avec des Permissions Élevées

**Commande :**
```bash
docker run --rm --privileged alpine sh -c 'echo hello from privileged mode'
```
**Objectif :** Observer pourquoi l’exécution d’un conteneur en mode `--privileged` est dangereuse. 

 **Risques identifiés :**
- Accès total aux ressources de l’hôte.
- Possibilité de modifier les fichiers critiques du système.
- Escalade de privilèges et compromission de la machine hôte.
![ok](https://github.com/Ozankrd/container-security/blob/s1/1.png)
---

##  2. Simuler une Évasion de Conteneur

🔹 **Commande :**
```bash
docker run --rm -v /:/mnt alpine sh -c 'ls /mnt'
```
🔹 **Objectif :** Vérifier si le conteneur peut accéder au système de fichiers de l’hôte.

**Risques identifiés :**
- Accès en lecture/écriture aux fichiers sensibles de l’hôte.
- Possibilité de modifier les configurations critiques du système.
- Menace d’exécution de commandes malveillantes sur l’hôte.

![ok](https://github.com/Ozankrd/container-security/blob/s1/1.png)

---

## 3. Créer une Image Sécurisée

🔹 **Dockerfile Minimaliste :**
```dockerfile
FROM alpine
RUN adduser -D appuser
USER appuser
CMD ["echo", "Container sécurisé!"]
```
🔹 **Étapes :**
1. Construire l’image :
   ```bash
   docker build -t secure-container .
   ```
2. Exécuter le conteneur :
   ```bash
   docker run --rm secure-container
   ```
3. Vérifier l’ID et l’UID de l’utilisateur :
   ```bash
   docker run --rm secure-container id
   ```

**Bonnes pratiques appliquées :**
- Création d’un utilisateur non-root pour limiter les privilèges.
- Utilisation d’une image légère (`alpine`) pour réduire la surface d’attaque.
- 
![ok](https://github.com/Ozankrd/container-security/blob/s1/3.png)

---

## 4. Restreindre l’Accès Réseau d’un Conteneur

🔹 **Déconnecter le réseau du conteneur :**
```bash
docker network disconnect bridge mon-container
```
🔹 **Tester l’accès internet :**
```bash
docker exec -it mon-container ping -c 4 google.com
```
**Résultat attendu :** Le ping échoue, prouvant que le conteneur n’a plus accès à Internet.
![ok](https://github.com/Ozankrd/container-security/blob/s1/4.png)

---

## 5. Télécharger et Scanner une Image avec Trivy

🔹 **Télécharger une image vulnérable et l’analyser :**
```bash
docker pull vulnerables/web-dvwa
trivy image vulnerables/web-dvwa
```
🔹 **Sauvegarder le résultat en JSON :**
```bash
trivy image -f json -o scan_result.json vulnerables/web-dvwa
```
**Résumé des vulnérabilités :**
- Détection d’une clé privée (`/etc/ssl/private/ssl-cert-snakeoil.key`).
- Plusieurs failles critiques sur des paquets obsolètes.
- Risques liés aux composants non mis à jour.
![ok](https://github.com/Ozankrd/container-security/blob/s1/56.png)

---

## 6. Scanner une Image pour Détecter les Vulnérabilités avec Grype

🔹 **Scanner une image avec Grype :**
```bash
grype alpine:latest
```
🔹 **Comparer avec une image buildée :**
```bash
grype mon-image:latest > scan_custom_image.txt
grype alpine:latest > scan_alpine.txt
diff scan_custom_image.txt scan_alpine.txt
```
**Comparaison entre Grype et Trivy :**
| Outil  | Source des vulnérabilités | Rapidité | Précision |
|--------|-------------------------|----------|----------|
| **Grype** | Anchore Grype DB | Plus lent | Plus précis sur les paquets |
| **Trivy** | NVD, GitHub Advisory | Plus rapide | Peut générer des faux positifs |

**Conclusion :** Grype est plus précis, mais Trivy est plus rapide et détecte aussi les secrets.
![ok](https://github.com/Ozankrd/container-security/blob/s1/7.png)
![ok](https://github.com/Ozankrd/container-security/blob/s1/8.png)


---

##  Conclusion Générale

🔹 Ce projet a permis de :
Comprendre les risques liés aux conteneurs Docker.
Tester des vulnérabilités et des attaques courantes.
Appliquer des solutions pour sécuriser les conteneurs.
Analyser des images avec Trivy et Grype pour détecter des failles.

**Recommandations :**
- **Ne jamais exécuter un conteneur en mode `--privileged`**.
- **Éviter les montages `-v /:/mnt`** qui exposent l’hôte.
- **Restreindre l’accès réseau des conteneurs sensibles**.
- **Toujours scanner les images Docker avant déploiement**.

---

 
