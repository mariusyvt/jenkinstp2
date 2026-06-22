# Compte rendu — Mise en place d'un environnement DevOps avec Jenkins

## Sommaire

1. [Mise en place de la machine virtuelle](#1-mise-en-place-de-la-machine-virtuelle)
2. [Installation de Docker Engine](#2-installation-de-docker-engine)
3. [Déploiement de Jenkins](#3-déploiement-de-jenkins)
4. [Pipelines Jenkins](#4-pipelines-jenkins)
5. [Intégration continue et déploiement continu](#5-intégration-continue-et-déploiement-continu)
6. [Fork du repository et Jenkinsfile](#6-fork-du-repository-et-jenkinsfile)
7. [Agent Jenkins](#7-agent-jenkins)
8. [Exposition de Jenkins avec ngrok](#8-exposition-de-jenkins-avec-ngrok)
9. [Webhook GitHub ↔ Jenkins](#9-webhook-github--jenkins)
10. [Difficultés rencontrées](#10-difficultés-rencontrées)

---

## 1. Mise en place de la machine virtuelle

> ⚠️ **Note** : la consigne demandait l'utilisation de VMWare, mais j'ai utilisé **VirtualBox** pour la création et la gestion de la machine virtuelle.

### Installation de VirtualBox

Téléchargement et installation de VirtualBox depuis le site officiel.

### Création de la machine virtuelle DevOps

Création d'une VM avec :

- Système : Ubuntu Server
- Utilisateur : `devops`
- Mode réseau : NAT avec redirection de port (port hôte `2222` → port invité `22` pour l'accès SSH)

### Vérification de l'accès internet et de l'adresse IP

Vérification depuis la VM que la machine dispose bien d'une adresse IP et d'un accès internet.

```bash
ip a
```

Adresse IP obtenue sur l'interface `enp0s3` : `10.0.2.15`

### Snapshot de la machine vierge

Réalisation d'un snapshot de la VM avant toute installation, afin de pouvoir revenir à un état propre en cas de problème.

---

## 2. Installation de Docker Engine

Installation de Docker Engine sur la machine DevOps en suivant la documentation officielle Docker.

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg
# ... installation du dépôt officiel Docker et de docker-ce
```

### Création du réseau Docker "devops"

```bash
docker network create devops
```

---

## 3. Déploiement de Jenkins

### Déploiement de l'image Jenkins

Lancement du conteneur Jenkins sur le réseau `devops`, avec exposition des ports nécessaires (8080 pour l'interface web, 50000 pour la communication avec les agents).

```bash
docker run -d --name jenkins --network devops -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

### Finalisation de l'installation de Jenkins

Récupération du mot de passe administrateur initial, installation des plugins suggérés et création du compte administrateur.

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

---

## 4. Pipelines Jenkins

### Pipeline "Hello World"

Création d'une première pipeline simple affichant un message, afin de valider le bon fonctionnement de Jenkins.

```groovy
pipeline {
    agent any
    stages {
        stage('Hello') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

---

## 5. Intégration continue et déploiement continu

### Pipeline avec étape d'intégration continue

Ajout d'une pipeline comportant les étapes classiques d'intégration continue (build, test).

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                echo 'Récupération du code...'
            }
        }
        stage('Build') {
            steps {
                sh 'echo "Build OK"'
            }
        }
        stage('Test') {
            steps {
                sh 'echo "Tests OK"'
            }
        }
    }
}
```

### Ajout de l'étape de déploiement continu

Ajout d'une étape `Deploy` à la pipeline précédente pour couvrir le déploiement continu.

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                echo 'Récupération du code...'
            }
        }
        stage('Build') {
            steps {
                sh 'echo "Build OK"'
            }
        }
        stage('Test') {
            steps {
                sh 'echo "Tests OK"'
            }
        }
        stage('Deploy') {
            steps {
                sh 'echo "Deploy OK"'
            }
        }
    }
}
```

---

## 6. Fork du repository et Jenkinsfile

Fork du repository GitHub source, puis ajout d'un fichier `Jenkinsfile` à la racine du repository forké, reprenant les étapes de la pipeline CI/CD (Checkout, Build, Test, Deploy).

Repository utilisé : `https://github.com/mariusyvt/jenkinstp2`

Création d'un nouveau job Jenkins de type "Pipeline" configuré pour récupérer le Jenkinsfile depuis ce repository (option "Pipeline script from SCM").

---

## 7. Agent Jenkins

### Connexion de l'agent

Création d'un nouveau nœud (agent) `agent-1` depuis l'interface Jenkins, puis connexion de l'agent depuis la machine DevOps via JNLP :

```bash
java -jar agent.jar -url http://10.0.2.15:8080/ -secret <secret> -name "agent-1" -workDir "/home/devops/jenkins-agent"
```

> 💡 **Point d'attention** : le champ _"Remote root directory"_ de l'agent devait être aligné avec un dossier appartenant à l'utilisateur `devops` (`/home/devops/jenkins-agent`) pour éviter une erreur de permission (`AccessDeniedException`) lors du checkout Git.

### Lancement d'une pipeline sur l'agent

Modification du Jenkinsfile pour cibler l'agent :

```groovy
pipeline {
    agent { label 'agent-1' }
    ...
}
```

Lancement du build et vérification dans la console que l'exécution se fait bien sur `agent-1`.

---

## 8. Exposition de Jenkins avec ngrok

Installation de ngrok sur la machine DevOps, configuration de l'authtoken, puis création d'un tunnel vers le port 8080 de Jenkins afin de le rendre accessible temporairement depuis internet.

```bash
ngrok config add-authtoken <token>
nohup ngrok http 8080 > ngrok.log 2>&1 &
```

URL publique générée : `https://xxxx-xx-xx-xxx-xx.ngrok-free.app`

---

## 9. Webhook GitHub ↔ Jenkins

### Configuration du webhook sur GitHub

Ajout d'un webhook sur le repository forké, pointant vers l'URL ngrok :

```
https://xxxx-xx-xx-xxx-xx.ngrok-free.app/github-webhook/
```

### Configuration du déclencheur sur Jenkins

Activation de l'option **"GitHub hook trigger for GITScm polling"** dans les déclencheurs de construction du job Jenkins.

### Test du déclenchement automatique

Réalisation d'un commit/push sur le repository forké, et vérification que la pipeline Jenkins se déclenche automatiquement sans intervention manuelle.

---

## 10. Difficultés rencontrées

| Problème                                                                | Cause                                                                                                                        | Solution                                                                                                                       |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Agent déconnecté automatiquement par Jenkins                            | `/tmp` monté en `tmpfs` limité à ~822 Mio                                                                                    | Désactivation du `tmpfs` sur `/tmp` (`systemctl mask tmp.mount` + reboot) pour utiliser l'espace disque de la partition racine |
| `Connection refused` lors de la connexion de l'agent                    | Conteneur Jenkins arrêté après le reboot de la VM                                                                            | Redémarrage du conteneur avec `docker start jenkins`                                                                           |
| `AccessDeniedException: /home/jenkins` lors du checkout Git sur l'agent | Le champ _"Remote root directory"_ de l'agent pointait vers un dossier non accessible en écriture par l'utilisateur `devops` | Modification du _"Remote root directory"_ vers `/home/devops/jenkins-agent` et redémarrage de l'agent                          |

---

## Conclusion

L'ensemble de la chaîne CI/CD a été mise en place avec succès : de la création de la machine virtuelle jusqu'au déclenchement automatique des pipelines Jenkins via webhook GitHub, en passant par la configuration d'un agent dédié et l'exposition temporaire de Jenkins via ngrok.
