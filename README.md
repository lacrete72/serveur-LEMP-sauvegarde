# serveur-LEMP-sauvegarde

ajouter une ip sur pour que notre carte ai 2 ip.
```
sudo nano /etc/network/interfaces
```

ajouter cette ligne 
```
up ip addr add <IP/MASK> dev enp0s3
```

Générer une clé SSH

```
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa
```

envoyé la clé public vers le serveur 
```
ssh-copy-id -i ~/.ssh/id_rsa.pub ton_user@sftp.example.com
```

création d'un script
```
#!/bin/bash

# Variables à personnaliser
SFTP_USER="pierre"               # Utilisateur SFTP
SFTP_HOST="10.200.0.1"           # Adresse IP du serveur SFTP
SFTP_PATH="/home/pierre/backups" # Dossier distant où les sauvegardes seront stockées
DISCORD_WEBHOOK_URL="https://discord.com/api/webhooks/1436014874143625246/tiEIo8eTmwwmDD3mEcgAYQzvYc3woIthcZyXnIsvzikAVY6kYxbLqFHO_ogv_hYuMBc1"  # URL du webhook Discord
LOCAL_BACKUP_DIR="/home/debian/backup" # Répertoire local pour stocker les backups avant envoi
WORDPRESS_DIR="/home/debian/wordpress_data"    # Dossier WordPress à sauvegarder
DOCKER_COMPOSE_PATH="/home/debian/docker"
DB_USER="debian"                 # Utilisateur de la base de données MySQL
DB_PASSWORD="root"               # Mot de passe MySQL
DB_NAME="wordpress_db"           # Nom de la base de données à sauvegarder
DATE=$(date +%F_%H-%M-%S)        # Date et heure pour nommer les fichiers de sauvegarde
LOG_FILE="/home/debian/backup.log"   # Log de la sauvegarde

# Fonction pour envoyer un message sur Discord
send_discord_message() {
    local message=$1
    curl -X POST -H "Content-Type: application/json" \
    -d "{\"content\":\"$message\"}" \
    $DISCORD_WEBHOOK_URL
}

# Création du répertoire local de sauvegarde s'il n'existe pas
mkdir -p $LOCAL_BACKUP_DIR

# Fonction pour afficher les messages à la fois sur le terminal et dans le log
log_message() {
    echo "$(date) : $1" | tee -a $LOG_FILE
}

# Activer le mode débogage pour afficher les commandes
set -x

# Faire un dump de la base de données
log_message "Démarrage du dump de la base de données"
docker compose -f $DOCKER_COMPOSE_PATH/docker-compose.yml exec db mysqldump -u $DB_USER -p$DB_PASSWORD --no-tablespaces $DB_NAME > $LOCAL_BACKUP_DIR/${DB_NAME}_$DATE.sql

if [[ $? -ne 0 ]]; then
    log_message "ERREUR - Échec du dump de la base de données"
    send_discord_message "Échec du dump de la base de données pour $DB_NAME."
    exit 1
else
    log_message "Dump de la base de données réussi"
fi

# Sauvegarder les fichiers WordPress
log_message "Démarrage de la sauvegarde des fichiers WordPress"
tar czf $LOCAL_BACKUP_DIR/wordpress_files_$DATE.tar.gz -C $WORDPRESS_DIR . 2>> $LOG_FILE

if [[ $? -ne 0 ]]; then
    log_message "ERREUR - Échec de la sauvegarde des fichiers WordPress"
    send_discord_message "Échec de la sauvegarde des fichiers WordPress."
    exit 1
else
    log_message "Sauvegarde des fichiers WordPress réussie"
fi

# Vérification de l'intégrité des fichiers (base de données et archives)
log_message "Vérification de l'intégrité des fichiers"

# Vérifier si les fichiers existent localement
if [[ ! -f $LOCAL_BACKUP_DIR/${DB_NAME}_$DATE.sql || ! -f $LOCAL_BACKUP_DIR/wordpress_files_$DATE.tar.gz ]]; then
    log_message "ERREUR - Les fichiers de sauvegarde sont manquants"
    send_discord_message "Les fichiers de sauvegarde sont manquants."
    exit 1
else
    log_message "Les fichiers de sauvegarde sont prêts à être envoyés"
fi

# Transférer les fichiers via SFTP
log_message "Transfert des fichiers vers le serveur SFTP"
sftp $SFTP_USER@$SFTP_HOST <<EOF
mkdir -p $SFTP_PATH/$DATE
put $LOCAL_BACKUP_DIR/${DB_NAME}_$DATE.sql $SFTP_PATH/$DATE/
put $LOCAL_BACKUP_DIR/wordpress_files_$DATE.tar.gz $SFTP_PATH/$DATE/
bye
EOF

if [[ $? -ne 0 ]]; then
    log_message "ERREUR - Problème lors de la copie des fichiers vers le serveur SFTP"
    send_discord_message "Erreur lors de la copie des fichiers de sauvegarde vers le serveur SFTP."
    exit 1
else
    log_message "Sauvegarde envoyée avec succès vers le serveur SFTP"
fi

# Nettoyage des fichiers locaux après transfert
rm -f $LOCAL_BACKUP_DIR/${DB_NAME}_$DATE.sql
rm -f $LOCAL_BACKUP_DIR/wordpress_files_$DATE.tar.gz

# Envoi d'un message Discord avec le statut de la sauvegarde
send_discord_message "La sauvegarde du site WordPress a été effectuée avec succès !"

# Log de fin de processus
log_message "Sauvegarde terminée"

# Désactiver le mode débogage
set +x


```

