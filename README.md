# Mise en place de la sauvegarde automatisée pour Nextcloud avec Docker Compose

## 1. Préparation des répertoires

### a) Créer le répertoire `toip` et le fichier CSV
Connectez-vous à votre serveur Debian 12 et exécutez les commandes suivantes pour créer le répertoire `toip` et le fichier `data.csv` vide :
```bash
mkdir -p /srv/nextcloud/toip
touch /srv/nextcloud/toip/data.csv
```

### b) Créer le répertoire `archive` pour les sauvegardes
Ensuite, créez un répertoire `archive` pour stocker les sauvegardes compressées :
```bash
mkdir -p /srv/nextcloud/archive
```

---

## 2. Écriture du script Bash `backup_toip.sh`

### a) Créer le script de sauvegarde
Créez un fichier nommé `backup_toip.sh` dans un répertoire comme `/srv/nextcloud/scripts/` :

```bash
#!/bin/bash

# Variables
TOIP_DIR="/srv/nextcloud/toip"
ARCHIVE_DIR="/srv/nextcloud/archive"
FTP_SERVER="ftp://192.168.20.112"
FTP_USER="toto"
FTP_PASS="123"
FTP_DIR="archives_toip"

# Obtenir la date actuelle
DATE=$(date +'%d-%m-%Y_%H:%M:%S')

# Sauvegarde du fichier CSV
CSV_FILE="sio2-$DATE.csv"
cp "$TOIP_DIR/data.csv" "$ARCHIVE_DIR/$CSV_FILE"

# Compression du répertoire `toip`
ZIP_FILE="sio2-$DATE.zip"
zip -r "$ARCHIVE_DIR/$ZIP_FILE" "$TOIP_DIR"

# Transfert du fichier ZIP sur le serveur FTP
curl -T "$ARCHIVE_DIR/$ZIP_FILE" --user "$FTP_USER:$FTP_PASS" "$FTP_SERVER/$FTP_DIR/"

# Nettoyer les fichiers compressés locaux (facultatif : garde les fichiers 7 jours)
find "$ARCHIVE_DIR" -type f -mtime +7 -exec rm {} \;
```

### b) Rendre le script exécutable
Exécutez cette commande pour rendre le script exécutable :
```bash
chmod +x /srv/nextcloud/scripts/backup_toip.sh
```

---

## 3. Automatisation avec Cron

### a) Ajouter une tâche cron
Ouvrez l'éditeur `crontab` pour ajouter une tâche qui exécutera le script tous les jours à 23:45 :
```bash
crontab -e
```

Ajoutez cette ligne pour planifier l'exécution du script :
```bash
45 23 * * * /srv/nextcloud/scripts/backup_toip.sh
```

Enregistrez et quittez l'éditeur.

---

## 4. Accès au répertoire Nextcloud dans Docker

### a) Modifier le fichier `docker-compose.yml`
Si le répertoire `toip` doit être accessible dans le conteneur Nextcloud, vous devez monter ce répertoire dans votre fichier `docker-compose.yml`. Voici un exemple de modification :
```yaml
services:
  nextcloud:
    image: nextcloud
    volumes:
      - /srv/nextcloud/toip:/var/www/html/toip
      - /srv/nextcloud/data:/var/www/html/data
    ...
```

### b) Appliquer les changements
Après avoir modifié le fichier `docker-compose.yml`, appliquez les changements en recréant le conteneur Docker :
```bash
docker-compose down
docker-compose up -d
```

---

## 5. Vérification

### a) Tester manuellement le script
Exécutez le script manuellement pour vérifier son bon fonctionnement :
```bash
/srv/nextcloud/scripts/backup_toip.sh
```

Vérifiez que :
- Le fichier `.csv` est sauvegardé dans `/srv/nextcloud/archive` avec le bon format de nom.
- Le répertoire `toip` est compressé.
- Le fichier ZIP est transféré sur le serveur FTP.

### b) Vérifier les journaux Docker en cas de problème
Si vous rencontrez des problèmes, consultez les journaux Docker du conteneur Nextcloud :
```bash
docker logs nextcloud
```

---
