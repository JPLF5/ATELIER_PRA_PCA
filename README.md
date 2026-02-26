------------------------------------------------------------------------------------------------------
ATELIER PRA/PCA
------------------------------------------------------------------------------------------------------
L’idée en 30 secondes : Cet atelier met en œuvre un **mini-PRA** sur **Kubernetes** en déployant une **application Flask** avec une **base SQLite** stockée sur un **volume persistant (PVC pra-data)** et des **sauvegardes automatiques réalisées chaque minute vers un second volume (PVC pra-backup)** via un **CronJob**. L’**image applicative est construite avec Packer** et le **déploiement orchestré avec Ansible**, tandis que Kubernetes assure la gestion des pods et de la disponibilité applicative. Nous observerons la différence entre **disponibilité** (recréation automatique des pods sans perte de données) et **reprise après sinistre** (perte volontaire du volume de données puis restauration depuis les backups), nous mesurerons concrètement les RTO et RPO, et comprendrons les limites d’un PRA local non répliqué. Cet atelier illustre de manière pratique les principes de continuité et de reprise d’activité, ainsi que le rôle respectif des conteneurs, du stockage persistant et des mécanismes de sauvegarde.
  
**Architecture cible :** Ci-dessous, voici l'architecture cible souhaitée.   
  
![Screenshot Actions](Architecture_cible.png)  
  
-------------------------------------------------------------------------------------------------------
Séquence 1 : Codespace de Github
-------------------------------------------------------------------------------------------------------
Objectif : Création d'un Codespace Github  
Difficulté : Très facile (~5 minutes)
-------------------------------------------------------------------------------------------------------
**Faites un Fork de ce projet**. Si besoin, voici une vidéo d'accompagnement pour vous aider à "Forker" un Repository Github : [Forker ce projet](https://youtu.be/p33-7XQ29zQ) 
  
Ensuite depuis l'onglet **[CODE]** de votre nouveau Repository, **ouvrez un Codespace Github**.
  
---------------------------------------------------
Séquence 2 : Création du votre environnement de travail
---------------------------------------------------
Objectif : Créer votre environnement de travail  
Difficulté : Simple (~10 minutes)
---------------------------------------------------
Vous allez dans cette séquence mettre en place un cluster Kubernetes K3d contenant un master et 2 workers, installer les logiciels Packer et Ansible. Depuis le terminal de votre Codespace copier/coller les codes ci-dessous étape par étape :  

**Création du cluster K3d**  
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
```
k3d cluster create pra \
  --servers 1 \
  --agents 2
```
**vérification de la création de votre cluster Kubernetes**  
```
kubectl get nodes
```
**Installation du logiciel Packer (création d'images Docker)**  
```
PACKER_VERSION=1.11.2
curl -fsSL -o /tmp/packer.zip \
  "https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip"
sudo unzip -o /tmp/packer.zip -d /usr/local/bin
rm -f /tmp/packer.zip
```
**Installation du logiciel Ansible**  
```
python3 -m pip install --user ansible kubernetes PyYAML jinja2
export PATH="$HOME/.local/bin:$PATH"
ansible-galaxy collection install kubernetes.core
```
  
---------------------------------------------------
Séquence 3 : Déploiement de l'infrastructure
---------------------------------------------------
Objectif : Déployer l'infrastructure sur le cluster Kubernetes
Difficulté : Facile (~15 minutes)
---------------------------------------------------  
Nous allons à présent déployer notre infrastructure sur Kubernetes. C'est à dire, créér l'image Docker de notre application Flask avec Packer, déposer l'image dans le cluster Kubernetes et enfin déployer l'infratructure avec Ansible (Création du pod, création des PVC et les scripts des sauvegardes aututomatiques).  

**Création de l'image Docker avec Packer**  
```
packer init .
packer build -var "image_tag=1.0" .
docker images | head
```
  
**Import de l'image Docker dans le cluster Kubernetes**  
```
k3d image import pra/flask-sqlite:1.0 -c pra
```
  
**Déploiment de l'infrastructure dans Kubernetes**  
```
ansible-playbook ansible/playbook.yml
```
  
**Forward du port 8080 qui est le port d'exposition de votre application Flask**  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
  
---------------------------------------------------  
**Réccupération de l'URL de votre application Flask**. Votre application Flask est déployée sur le cluster K3d. Pour obtenir votre URL cliquez sur l'onglet **[PORTS]** dans votre Codespace (à coté de Terminal) et rendez public votre port 8080 (Visibilité du port). Ouvrez l'URL dans votre navigateur et c'est terminé.  

**Les routes** à votre disposition sont les suivantes :  
1. https://...**/** affichera dans votre navigateur "Bonjour tout le monde !".
2. https://...**/health** pour voir l'état de santé de votre application.
3. https://...**/add?message=test** pour ajouter un message dans votre base de données SQLite.
4. https://...**/count** pour afficher le nombre de messages stockés dans votre base de données SQLite.
5. https://...**/consultation** pour afficher les messages stockés dans votre base de données.
  
---------------------------------------------------  
### Processus de sauvegarde de la BDD SQLite

Grâce à une tâche CRON déployée par Ansible sur le cluster Kubernetes (un CronJob), toutes les minutes une sauvegarde de la BDD SQLite est faite depuis le PVC pra-data vers le PCV pra-backup dans Kubernetes.  

Pour visualiser les sauvegardes périodiques déposées dans le PVC pra-backup, coller les commandes suivantes dans votre terminal Codespace :  

```
kubectl -n pra run debug-backup \
  --rm -it \
  --image=alpine \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "debug",
      "image": "alpine",
      "command": ["sh"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "name": "backup",
        "mountPath": "/backup"
      }]
    }],
    "volumes": [{
      "name": "backup",
      "persistentVolumeClaim": {
        "claimName": "pra-backup"
      }
    }]
  }
}'
```
```
ls -lh /backup
```
**Pour sortir du cluster et revenir dans le terminal**
```
exit
```

---------------------------------------------------
Séquence 4 : 💥 Scénarios de crash possibles  
Difficulté : Facile (~30 minutes)
---------------------------------------------------
### 🎬 **Scénario 1 : PCA — Crash du pod**  
Nous allons dans ce scénario **détruire notre Pod Kubernetes**. Ceci simulera par exemple la supression d'un pod accidentellement, ou un pod qui crash, ou un pod redémarré, etc..

**Destruction du pod :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario1.png)  

Nous perdons donc ici notre application mais pas notre base de données puisque celle-ci est déposée dans le PVC pra-data hors du pod.  

Copier/coller le code suivant dans votre terminal Codespace pour détruire votre pod :
```
kubectl -n pra get pods
```
Notez le nom de votre pod qui est différent pour tout le monde.  
Supprimez votre pod (pensez à remplacer <nom-du-pod-flask> par le nom de votre pod).  
Exemple : kubectl -n pra delete pod flask-7c4fd76955-abcde  
```
kubectl -n pra delete pod <nom-du-pod-flask>
```
**Vérification de la suppression de votre pod**
```
kubectl -n pra get pods
```
👉 **Le pod a été reconstruit sous un autre identifiant**.  
Forward du port 8080 du nouveau service  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
Observez le résultat en ligne  
https://...**/consultation** -> Vous n'avez perdu aucun message.
  
👉 Kubernetes gère tout seul : Aucun impact sur les données ou sur votre service (PVC conserve la DB et le pod est reconstruit automatiquement) -> **C'est du PCA**. Tout est automatique et il n'y a aucune rupture de service.
  
---------------------------------------------------
### 🎬 **Scénario 2 : PRA - Perte du PVC pra-data** 
Nous allons dans ce scénario **détruire notre PVC pra-data**. C'est à dire nous allons suprimer la base de données en production. Ceci simulera par exemple la corruption de la BDD SQLite, le disque du node perdu, une erreur humaine, etc. 💥 Impact : IL s'agit ici d'un impact important puisque **la BDD est perdue**.  

**Destruction du PVC pra-data :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario2.png)  

🔥 **PHASE 1 — Simuler le sinistre (perte de la BDD de production)**  
Copier/coller le code suivant dans votre terminal Codespace pour détruire votre base de données :
```
kubectl -n pra scale deployment flask --replicas=0
```
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'
```
```
kubectl -n pra delete job --all
```
```
kubectl -n pra delete pvc pra-data
```
👉 Vous pouvez vérifier votre application en ligne, la base de données est détruite et la service n'est plus accéssible.  

✅ **PHASE 2 — Procédure de restauration**  
Recréer l’infrastructure avec un PVC pra-data vide.  
```
kubectl apply -f k8s/
```
Vérification de votre application en ligne.  
Forward du port 8080 du service pour tester l'application en ligne.  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
https://...**/count** -> =0.  
https://...**/consultation** Vous avez perdu tous vos messages.  

Retaurez votre BDD depuis le PVC Backup.  
```
kubectl apply -f pra/50-job-restore.yaml
```
👉 Vous pouvez vérifier votre application en ligne, **votre base de données a été restaureé** et tous vos messages sont bien présents.  

Relance des CRON de sauvgardes.  
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'
```
👉 Nous n'avons pas perdu de données mais Kubernetes ne gère pas la restauration tout seul. Nous avons du protéger nos données via des sauvegardes régulières (du PVC pra-data vers le PVC pra-backup). -> **C'est du PRA**. Il s'agit d'une stratégie de sauvegarde avec une procédure de restauration.  

---------------------------------------------------
Séquence 5 : Exercices  
Difficulté : Moyenne (~45 minutes)
---------------------------------------------------
**Complétez et documentez ce fichier README.md** pour répondre aux questions des exercices.  
Faites preuve de pédagogie et soyez clair dans vos explications et procedures de travail.  

**Exercice 1 :**  
Quels sont les composants dont la perte entraîne une perte de données ?  
  
Les composants dont la perte entraîne une perte de données sont les **PVC (Persistent Volume Claims)** :  
- **PVC `pra-data`** : contient la base de données SQLite de production (`app.db`). Sa perte entraîne la **perte immédiate de toutes les données applicatives**. C'est ce que nous avons simulé dans le Scénario 2.  
- **PVC `pra-backup`** : contient les sauvegardes horodatées de la BDD. Sa perte entraînerait l'**impossibilité de restaurer les données** en cas de sinistre sur le PVC `pra-data`.  

En revanche, la perte d'un **pod** (Scénario 1) n'entraîne **aucune perte de données** car les pods sont éphémères et les données sont stockées sur les PVC, qui sont indépendants du cycle de vie des pods.

**Exercice 2 :**  
Expliquez nous pourquoi nous n'avons pas perdu les données lors de la supression du PVC pra-data  
  
Lors de la suppression du PVC `pra-data` (Scénario 2), nous n'avons pas perdu définitivement les données grâce au **mécanisme de sauvegarde automatique** mis en place via un **CronJob Kubernetes** (`sqlite-backup`).  

Ce CronJob s'exécute **toutes les minutes** et copie le fichier `app.db` depuis le PVC `pra-data` vers le PVC `pra-backup` avec un horodatage (timestamp Unix). Ainsi, même après la destruction du PVC `pra-data`, les copies de sauvegarde restaient intactes dans le PVC `pra-backup`.  

La procédure de restauration a consisté à :  
1. Recréer un PVC `pra-data` vide via `kubectl apply -f k8s/`  
2. Lancer le job de restauration (`pra/50-job-restore.yaml`) qui copie la **sauvegarde la plus récente** depuis `/backup` vers `/data/app.db`  

C'est le principe fondamental du **PRA** : les données sont protégées par des sauvegardes régulières stockées sur un volume séparé, permettant une restauration en cas de sinistre.

**Exercice 3 :**  
Quels sont les RTO et RPO de cette solution ?  
  
**RPO (Recovery Point Objective)** — Perte de données maximale tolérée :  
Le RPO est de **1 minute maximum**. En effet, le CronJob de sauvegarde s'exécute toutes les minutes (`*/1 * * * *`). Dans le pire des cas, les données écrites dans la dernière minute avant le sinistre seront perdues (celles qui n'ont pas encore été sauvegardées).  

**RTO (Recovery Time Objective)** — Temps de reprise du service :  
Le RTO est estimé à **environ 2 à 5 minutes**. Il correspond au temps nécessaire pour :  
1. Détecter le sinistre (~variable)  
2. Recréer le PVC `pra-data` vide via `kubectl apply -f k8s/` (~30 secondes)  
3. Lancer le job de restauration via `kubectl apply -f pra/50-job-restore.yaml` (~30 secondes)  
4. Vérifier que l'application est de nouveau fonctionnelle (~30 secondes)  
5. Réactiver le CronJob de sauvegarde (~10 secondes)  

⚠️ Ce RTO suppose une **intervention humaine** pour détecter le problème et exécuter la procédure de restauration. Sans monitoring ni automatisation de la détection, le RTO réel peut être bien plus élevé.

**Exercice 4 :**  
Pourquoi cette solution (cet atelier) ne peux pas être utilisé dans un vrai environnement de production ? Que manque-t-il ?   
  
Cette solution présente plusieurs **limites** qui la rendent inadaptée à un environnement de production :  

1. **Pas de réplication géographique** : Les PVC `pra-data` et `pra-backup` sont sur le **même cluster** (même machine physique). En cas de panne matérielle du serveur, les deux volumes sont perdus simultanément.  
2. **Base de données SQLite** : SQLite n'est pas conçu pour des accès concurrents multiples. En production, il faudrait utiliser une base de données client-serveur (PostgreSQL, MySQL) avec réplication.  
3. **Pas de monitoring ni d'alerting** : Aucun système de surveillance ne détecte automatiquement les pannes. La détection repose sur une intervention humaine, ce qui augmente le RTO réel.  
4. **Restauration manuelle** : La procédure de restauration nécessite une intervention humaine (lancer les commandes kubectl). Il n'y a pas de failover automatique.  
5. **Pas de rétention des backups** : Les sauvegardes s'accumulent indéfiniment dans le PVC `pra-backup` sans politique de rotation ou de nettoyage, ce qui finira par saturer le stockage.  
6. **Cluster mono-nœud maître** : Il n'y a qu'un seul serveur K3d. La perte de ce nœud maître rend tout le cluster indisponible.  
7. **Pas de chiffrement des sauvegardes** : Les backups ne sont pas chiffrés, ce qui pose un problème de sécurité des données.  
8. **Pas de test automatisé du PRA** : La procédure de restauration n'est jamais testée automatiquement, on ne peut pas garantir qu'elle fonctionnera le jour J.
  
**Exercice 5 :**  
Proposez une archtecture plus robuste.   
  
Voici une architecture de production plus robuste :  

**1. Base de données** : Remplacer SQLite par **PostgreSQL** déployé avec l'opérateur **CloudNativePG** ou **Zalando Postgres Operator**, offrant la réplication synchrone, le failover automatique et les sauvegardes intégrées (WAL archiving).  

**2. Cluster Kubernetes multi-nœuds et multi-zones** : Déployer un cluster Kubernetes managé (EKS, GKE, AKS) sur **au moins 3 zones de disponibilité** avec plusieurs nœuds maîtres (control plane HA) pour survivre à la perte d'une zone entière.  

**3. Stockage répliqué** : Utiliser un système de stockage distribué comme **Longhorn** ou **Rook-Ceph** pour assurer la réplication des données sur plusieurs nœuds, ou utiliser les classes de stockage managées du cloud provider (gp3 sur AWS, pd-ssd sur GCP).  

**4. Sauvegardes distantes (off-site)** : Exporter les sauvegardes vers un stockage objet distant (AWS S3, GCS, Azure Blob) avec **Velero** ou un outil dédié. Appliquer une politique de rétention (ex. : 7 jours de backups horaires, 4 semaines de backups quotidiens).  

**5. Monitoring et alerting** : Déployer **Prometheus + Grafana** pour la surveillance et configurer des alertes automatiques (PagerDuty, Slack) en cas de panne détectée.  

**6. Failover automatique** : Mettre en place un mécanisme de restauration automatique déclenché par les alertes, réduisant le RTO à quelques secondes.  

**7. Tests PRA réguliers** : Planifier des **exercices de reprise** périodiques (chaos engineering avec LitmusChaos ou ChaosMonkey) pour valider que la procédure fonctionne.  

**8. Chiffrement** : Chiffrer les sauvegardes au repos et en transit, et utiliser des secrets Kubernetes managés (Sealed Secrets, Vault) pour les credentials.

---------------------------------------------------
Séquence 6 : Ateliers  
Difficulté : Moyenne (~2 heures)
---------------------------------------------------
### **Atelier 1 : Ajoutez une fonctionnalité à votre application**  
**Ajouter une route GET /status** dans votre application qui affiche en JSON :
* count : nombre d’événements en base
* last_backup_file : nom du dernier backup présent dans /backup
* backup_age_seconds : âge du dernier backup

La route `GET /status` a été ajoutée dans `app/app.py`. Elle retourne un JSON contenant :  
- `count` : nombre d'événements en base  
- `last_backup_file` : nom du dernier fichier de backup présent dans `/backup`  
- `backup_age_seconds` : âge en secondes du dernier backup  

Le déploiement Kubernetes (`k8s/20-deployment.yaml`) a été modifié pour monter également le PVC `pra-backup` dans le pod Flask sur `/backup`, afin que la route `/status` puisse lire les fichiers de sauvegarde.  

*..**Déposez ici une copie d'écran** de votre réussite..*

---------------------------------------------------
### **Atelier 2 : Choisir notre point de restauration**  
Aujourd’hui nous restaurobs “le dernier backup”. Nous souhaitons **ajouter la capacité de choisir un point de restauration**.

#### Runbook — Restauration depuis un point de restauration choisi

**Contexte** : Par défaut, le job de restauration (`pra/50-job-restore.yaml`) restaure le dernier backup. Nous avons ajouté la possibilité de **choisir un point de restauration spécifique** grâce à un job paramétré (`pra/51-job-restore-targeted.yaml`).  

**Étape 1 — Lister les backups disponibles**  
```bash
kubectl -n pra run list-backups --rm -it --image=alpine \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "list",
        "image": "alpine",
        "command": ["ls", "-lht", "/backup"],
        "volumeMounts": [{
          "name": "backup",
          "mountPath": "/backup"
        }]
      }],
      "volumes": [{
        "name": "backup",
        "persistentVolumeClaim": {
          "claimName": "pra-backup"
        }
      }]
    }
  }'
```
Notez le nom du fichier de backup souhaité (ex : `app-1740500460.db`).  

**Étape 2 — Préparer la restauration (arrêter l'app et les sauvegardes)**  
```bash
kubectl -n pra scale deployment flask --replicas=0
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'
kubectl -n pra delete job --all
```

**Étape 3 — Lancer la restauration ciblée**  
Remplacez `<NOM_DU_BACKUP>` par le fichier choisi à l'étape 1 :  
```bash
kubectl -n pra delete job sqlite-restore-targeted --ignore-not-found
BACKUP_FILE="<NOM_DU_BACKUP>" envsubst < pra/51-job-restore-targeted.yaml | kubectl apply -f -
```
Ou plus simplement, éditez le fichier `pra/51-job-restore-targeted.yaml` en remplaçant `${BACKUP_FILE}` par le nom du fichier, puis :  
```bash
kubectl apply -f pra/51-job-restore-targeted.yaml
```

**Étape 4 — Vérifier la restauration et relancer les services**  
```bash
kubectl -n pra scale deployment flask --replicas=1
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
Vérifier via `/consultation` et `/count` que les données attendues sont bien présentes.  

**Étape 5 — Réactiver les sauvegardes**  
```bash
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'
```  
  
---------------------------------------------------
Evaluation
---------------------------------------------------
Cet atelier PRA PCA, **noté sur 20 points**, est évalué sur la base du barème suivant :  
- Série d'exerices (5 points)
- Atelier N°1 - Ajout d'un fonctionnalité (4 points)
- Atelier N°2 - Choisir son point de restauration (4 points)
- Qualité du Readme (lisibilité, erreur, ...) (3 points)
- Processus travail (quantité de commits, cohérence globale, interventions externes, ...) (4 points) 

