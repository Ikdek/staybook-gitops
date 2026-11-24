# Réponses — TP Migrer StayBook vers Kubernetes avec Argo CD

**Nom(s) :**  
**Dépôt GitHub :** https://github.com/Ikdek/staybook-gitops


## Étape 1 — Docker Compose : le point de départ

### Question 1

_Dans ce `docker-compose.yml`, repérez tout ce que Compose fait **pour vous** et que vous n'aurez plus « gratuitement » dans Kubernetes : l'ordre de démarrage, le réseau, les volumes, les ports, les variables d'environnement, la construction des images. Pour chacun, notez le mot-clé Compose correspondant._

Réponse :


### Question 2

_Pourquoi faut-il impérativement faire `docker compose down` avant de continuer ? (Deux raisons : l'une concerne un port, l'autre la mémoire.)_

Réponse :


## Étape 2 — Kubernetes + Argo CD : la migration

### Question 3

_En mode push, qui détient les droits d'administration sur le cluster ? Citez deux risques concrets que le modèle pull réduit._

Réponse : Le pipeline CI (il stocke un kubeconfig admin). Risques réduits : fuite des identifiants admin si la CI est compromise ; API du cluster exposée pour que la CI puisse la joindre.


### Question 4

_Un collègue modifie « à la main » le nombre de réplicas d'un déploiement avec `kubectl scale` sur un cluster géré par Argo CD. D'après le principe 4, que va-t-il se passer ? Est-ce souhaitable ?_

Réponse : Argo CD détecte l'écart (OutOfSync) et, avec `selfHeal`, remet la valeur de Git. Oui : Git est la seule source de vérité, toute modif doit passer par un commit tracé.


### Question 5

_Quel composant est le seul à parler à GitHub ? Lequel est le seul à créer des ressources dans le cluster ?_

Réponse : `argocd-repo-server` parle à GitHub ; `argocd-application-controller` crée les ressources.

| Composant | Rôle | Type |
|---|---|---|
| argocd-server | API + interface web | Deployment |
| argocd-repo-server | Clone Git, génère les manifests | Deployment |
| argocd-application-controller | Compare Git ↔ cluster et synchronise | StatefulSet |
| argocd-redis | Cache | Deployment |
| argocd-dex-server | SSO / authentification | Deployment |
| argocd-applicationset-controller | Génère des Applications depuis des modèles | Deployment |
| argocd-notifications-controller | Envoie des notifications | Deployment |


### Question 6

_Le cluster est « vierge ». Quels namespaces existent déjà, et à quoi servent-ils ? Combien de pods tournent, et pourquoi n'y en a-t-il aucun dans `default` ?_

Réponse : `default` (charges utilisateur), `kube-system` (composants du cluster), `kube-public` (infos publiques), `kube-node-lease` (heartbeats des nœuds), `local-path-storage` (provisionnement des volumes, kind). 9 pods. Aucun dans `default` car ce sont des composants système et rien n'est encore déployé.


### Question 7

_Pourquoi le script tague les images `v1` plutôt que `latest` ? Cherchez la règle de Kubernetes sur `imagePullPolicy` quand le tag est `latest` : que se passerait-il au démarrage d'un pod ?_

Réponse : Avec `latest`, `imagePullPolicy` vaut `Always` : le kubelet chercherait l'image sur Docker Hub, ne la trouverait pas → `ImagePullBackOff`. Avec `v1`, c'est `IfNotPresent` : l'image chargée localement est utilisée. Un tag versionné est aussi traçable.


### Question 8

_Où sont physiquement les images maintenant ? Vérifiez avec `docker exec argocd-tp-control-plane crictl images | grep staybook`. Qu'est-ce que `crictl`, et pourquoi n'est-ce pas `docker images` ?_

Réponse : Dans containerd, à l'intérieur du conteneur-nœud `argocd-tp-control-plane`. `crictl` est la CLI des runtimes CRI. Le nœud utilise containerd, pas Docker ; `docker images` montre le stockage de l'hôte.


### Question 9

_Combien d'objets vont être créés ? Parmi les types listés, trois ne sont pas des objets Kubernetes « standard ». Lesquels, et à quoi servent-ils ? (Indice : `CustomResourceDefinition`.)_

Réponse : 60 objets. Les 3 CRD ajoutent : `Application` (lien dépôt → cluster), `AppProject` (regroupe et restreint des applications), `ApplicationSet` (génère des Applications automatiquement).


### Question 10

_Pourquoi est-il préférable de passer par Kustomize plutôt que de faire `kubectl apply -f install.yaml` puis `kubectl patch` sur le Service ? Pensez au jour où vous mettrez Argo CD à jour._

Réponse : Le patch manuel n'est pas versionné et serait écrasé à la mise à jour. Avec Kustomize, les patchs sont dans Git et réappliqués automatiquement : mettre à jour = changer la version dans l'URL.


### Question 11

_Vérifiez votre tableau de la partie 1 : quel composant est un StatefulSet ? Avez-vous les sept composants ?_

Réponse : Oui, 7 composants. `argocd-application-controller` est le StatefulSet (identité stable pour le sharding).


### Question 12

_Essayez `kubectl apply -k argocd-install` **sans** `--server-side` (c'est sans danger, tout existe déjà). Quelle erreur obtenez-vous, sur quel objet ? Cherchez la cause (`last-applied-configuration`)._

Réponse : Erreur `metadata.annotations: Too long: must have at most 262144 bytes` sur la CRD `applicationsets.argoproj.io`. L'apply client copie tout l'objet dans l'annotation `last-applied-configuration`, qui dépasse 256 Ko. Le server-side apply n'utilise pas cette annotation.


### Question 13

_Pourquoi la commande se termine-t-elle par `| base64 -d` ? Que verriez-vous sans ? Un Secret Kubernetes est-il chiffré ?_

Réponse : Les valeurs d'un Secret sont encodées en base64 ; sans décodage on voit une chaîne illisible. Non, un Secret n'est pas chiffré : base64 est un simple encodage.


### Question 14

_Que signifient `--plaintext` et `--grpc-web` ? Que se passe-t-il sans `--plaintext` (essayez) ? Ces options seraient-elles acceptables en production ?_

Réponse : `--plaintext` : connexion sans TLS. `--grpc-web` : gRPC encapsulé dans du HTTP/1.1 (passe les proxys). Sans `--plaintext`, la poignée de main TLS échoue. En production : `--plaintext` non (identifiants en clair), `--grpc-web` oui.


### Question 15

_Regardez `apps/staybook/00-secrets.yaml` : les mots de passe PostgreSQL sont en clair dans Git. Est-ce acceptable ici ? Et en entreprise ? Citez une solution._

Réponse : Acceptable ici (TP local, dépôt privé). Pas en entreprise : visibles par tous et conservés dans l'historique Git. Solution : Sealed Secrets (ou External Secrets + Vault, SOPS).


### Question 16

_Notez l'erreur. Pourquoi GitHub répond-il « Repository not found » et non « accès refusé » ?_

Réponse : Erreur « repository not found ». GitHub masque l'existence des dépôts privés : répondre « accès refusé » confirmerait que le dépôt existe.


### Question 17

_Sous quelle forme Argo CD a-t-il stocké ces identifiants dans le cluster ? Trouvez l'objet avec `kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository` et listez ses clés (`-o jsonpath='{.data}'` puis décodez `url` et `username`, pas `password`)._

Réponse : Un Secret Kubernetes `repo-3086762773` (namespace `argocd`). Clés : `password`, `type`, `url`, `username`. `url` = `https://github.com/Ikdek/staybook-gitops.git`, `username` = `Ikdek`, `type` = `git`.


### Question 18

_Comparez trois façons de s'authentifier (token personnel, deploy key SSH, GitHub App) : portée, rotation, ce qui se passe quand la personne quitte l'entreprise. Laquelle recommanderiez-vous pour 200 dépôts ?_

Réponse :

| | Token personnel | Deploy key SSH | GitHub App |
|---|---|---|---|
| Portée | Dépôts de l'utilisateur | 1 dépôt | Dépôts choisis, droits fins |
| Rotation | Manuelle | Manuelle, n'expire pas | Automatique (jetons de 1 h) |
| Départ de la personne | Le token cesse de marcher | Aucun impact | Aucun impact |

Pour 200 dépôts : GitHub App (une seule identité d'organisation, au lieu de 200 deploy keys).


### Question 19

_Combien de ressources Argo CD compte-t-il déployer ? Retrouvez chacune dans les fichiers de `apps/staybook/`._

Réponse : 16 ressources : 2 Secrets (`00`), 2 ConfigMaps + 2 Services + 2 StatefulSets PostgreSQL (`10`, `11`), Service + Deployment pour hotels-service (`20`), reservations-service (`21`), gateway (`30`) et front (`40`).


### Question 20

_Les ressources ne sont pas déployées toutes en même temps. Ouvrez les manifests et trouvez l'annotation qui pilote cet ordre. Quel ordre a été choisi, et pourquoi ? À quoi cela correspondait-il dans `docker-compose.yml` ?_

Réponse : Annotation `argocd.argoproj.io/sync-wave`. Ordre : 0 bases et secrets → 1 hotels/reservations → 2 gateway → 3 front, chacun dépendant du précédent. Équivalent de `depends_on` dans Compose.


### Question 21

_« OutOfSync » et « Healthy » sont deux statuts indépendants. Donnez un exemple d'application `Synced` mais `Degraded`, et un exemple `OutOfSync` mais `Healthy`._

Réponse : Synced mais Degraded : Git demande l'image `v3` inexistante, appliquée mais en `ImagePullBackOff`. OutOfSync mais Healthy : `kubectl scale` manuel sans selfHeal, les pods tournent mais diffèrent de Git.


### Question 22

_Vous venez de faire un `kubectl apply`… n'est-ce pas contraire au GitOps ? Pourquoi ce dernier `apply` est-il inévitable, et comment réduire cette « amorce » à un seul objet pour tout un cluster ? (Cherchez « App of Apps ».)_

Réponse : Inévitable : Argo CD ne surveille un dépôt que via une Application, il faut donc créer la première (bootstrap). App of Apps : une seule Application racine pointe vers `argocd/`, qui contient toutes les autres → un seul `kubectl apply` pour tout le cluster.


### Question 23

_Reprenez votre réponse à la question 1 et complétez la colonne de droite en vous appuyant sur les manifests fournis._

Réponse :

| Ce que faisait Compose | Mot-clé Compose | Dans les manifests Kubernetes |
|---|---|---|
| Ordre de démarrage | `depends_on` | `sync-wave` |
| Attendre qu'un service soit prêt | `healthcheck` + `condition: service_healthy` | `readinessProbe`, `initContainers` |
| Réseau et DNS entre services | `networks` | `Service` (DNS du cluster) |
| Persistance des données | `volumes` | PVC / `volumeClaimTemplates` |
| Exposer le site | `ports` | Service `NodePort` 30081 + mapping kind |
| Variables d'environnement | `environment` | `env`, ConfigMap |
| Mots de passe | `environment` | `Secret` (`envFrom.secretRef`) |
| Image à utiliser | `build` / `image` | `image:` (préconstruite + `kind load`) |


### Question 24

_Pourquoi les deux Applications pointent-elles vers le **même namespace** `staybook` ? Que se passerait-il si le concierge était dans un namespace `concierge` ? (Indice : regardez `RAG_URL` dans le manifest de la gateway.)_

Réponse : `RAG_URL=http://rag-service:3003` est un nom court, résolu uniquement dans le même namespace. Dans `concierge`, la gateway ne trouverait plus le rag-service ; il faudrait `rag-service.concierge.svc.cluster.local`.


### Question 25

_Ouvrez `apps/concierge/20-rag-service.yaml`. Que font les deux `initContainers` ? En quoi est-ce différent des sync waves de la question 20 ? Pourquoi avoir les deux ?_

Réponse : Ils bouclent jusqu'à ce qu'Ollama et Qdrant répondent, avant de démarrer le rag-service. Les sync waves n'agissent qu'au sync d'Argo CD ; les initContainers agissent à chaque démarrage du pod. Les deux : ordre au déploiement + garantie à l'exécution (redémarrages).


### Question 26

_Ouvrez `apps/concierge/11-ollama.yaml`. Où sont stockés les modèles téléchargés ? Supprimez le pod Ollama (`kubectl -n staybook delete pod -l app=ollama`) et vérifiez avec `kubectl -n staybook exec deploy/ollama -- ollama list` que les modèles sont toujours là. Qu'est-ce qui les a conservés ? Que signifie `strategy: Recreate` ?_

Réponse : Dans `/root/.ollama`, monté sur le PVC `ollama-models`. Le PVC survit au pod, donc les modèles sont conservés. `Recreate` : l'ancien pod est supprimé avant de créer le nouveau (un seul pod sur le volume).


### Question 27

_Toujours dans ce fichier, lisez le bloc `resources`. Que se passe-t-il si Ollama dépasse la limite mémoire ? Et si le nœud n'a pas les 2 Gi demandés ?_

Réponse : Au-delà de 6 Gi : conteneur `OOMKilled` puis redémarré. Sans 2 Gi disponibles : pod `Pending` (`Insufficient memory`), il n'est pas planifié.


## Étape 3 — GitOps : Git pilote le cluster

### Question 28

_Sans `--refresh`, au bout de combien de temps Argo CD aurait-il vu le commit ? D'où vient cette valeur (ConfigMap `argocd-cm`, clé `timeout.reconciliation`) ? Quel mécanisme permettrait de réagir en quelques secondes sans interroger GitHub en boucle, et pourquoi ne peut-on pas l'utiliser ici ?_

Réponse : 3 minutes max (valeur par défaut de `timeout.reconciliation` = 180 s, non redéfinie dans `argocd-cm`). Un webhook GitHub réagirait en quelques secondes, mais GitHub ne peut pas joindre notre Argo CD sur `localhost`.


### Question 29

_Que s'est-il passé, et en combien de temps ? Quel composant a agi ? Retrouvez sa trace : `kubectl -n argocd logs sts/argocd-application-controller | grep staybook | tail`. Essayez plus fort : `kubectl -n staybook delete deploy gateway`. Le site est-il resté accessible ?_

Réponse : Le front est revenu à 1 réplica en quelques secondes, grâce à l'`argocd-application-controller` (selfHeal). La gateway supprimée a été recréée en quelques secondes : la page est restée affichée, l'API a été coupée brièvement.


### Question 30

_Décrivez précisément la chaîne d'événements entre votre `git push` et le nouveau titre dans le navigateur. Qui a fait quoi ? Où est passé l'ancien pod ? (`kubectl -n staybook get rs`)_

Réponse : `git push` → le repo-server lit le commit → l'application-controller voit l'écart et synchronise le Deployment → Kubernetes crée un nouveau ReplicaSet (`front-8df5f5754`, v2), le nouveau pod devient Ready → l'ancien pod est supprimé. L'ancien ReplicaSet (`front-758f88dcc5`) est conservé à 0 réplica, pour un éventuel rollback.


### Question 31

_Dans une entreprise, qui construirait et chargerait l'image à votre place, et où ? Quelle partie de ce que vous venez de faire relève de la **CI**, et quelle partie du **CD** ?_

Réponse : Un pipeline CI (GitHub Actions, GitLab CI) construit l'image et la pousse dans un registre (GHCR, Harbor…). CI : modifier le code, `docker build`, `kind load`. CD : changer le tag dans le dépôt GitOps, puis le sync par Argo CD.


### Question 32

_Qu'est-il arrivé au ConfigMap ? Que se serait-il passé avec `prune: false` ? Pourquoi `prune` est-il désactivé par défaut dans Argo CD ?_

Réponse : Supprimé du cluster automatiquement. Avec `prune: false`, il serait resté (application OutOfSync). Désactivé par défaut par sécurité : une erreur dans Git pourrait supprimer des ressources et des données de production.


### Question 33

_Après le rollback (a), l'auto-sync est-il toujours actif ? Pourquoi Argo CD fait-il cela ? Laquelle des deux approches est « GitOps-compatible » ? Le site est-il resté disponible pendant le mauvais déploiement, et grâce à quoi ?_

Réponse : Non, l'auto-sync est désactivé ; sinon Argo CD réappliquerait aussitôt la version `v3` de Git. L'approche GitOps-compatible est le `git revert`. Le site est resté disponible grâce au rolling update : le pod `v3` n'est jamais devenu Ready, donc l'ancien pod a continué à servir.


### Question 34

_Reliez chaque entrée de `argocd app history staybook` à un commit de `git log --oneline`. Face à un auditeur qui demande « qui a déployé quoi, quand », que répondez-vous, en combien de commandes ?_

Réponse :

| ID | Commit | Changement |
|---|---|---|
| 0 | `14ea210` | repoURL corrigé |
| 1 | `0ae8eab` | path corrigé, 1er déploiement |
| 2 | `6b26d39` | hotels-service à 2 réplicas |
| 3 | `8620509` | auto-sync activé |
| 4 | `733ccb4` | front v2 |
| 5 | `f2710bd` | ajout du ConfigMap de test |
| 6 | `21527cf` | suppression du ConfigMap |
| 7 | `c0adc2d` | front v3 (mauvaise image) |
| 8 | `c7dd7ba` | HEAD incluant le revert `d0e0b95` |

2 commandes : `argocd app history staybook` (quand) et `git log` (qui et quoi). Avec l'auto-sync, l'auteur du commit est celui qui a déployé.


## Preuves

- Sortie de `argocd app history staybook` :

```
SOURCE  https://github.com/Ikdek/staybook-gitops.git
ID      DATE                           REVISION
0       2026-11-24 13:32:37 +0100 CET  HEAD (14ea210)
1       2026-11-24 13:33:20 +0100 CET  HEAD (0ae8eab)
2       2026-11-24 13:43:11 +0100 CET  HEAD (6b26d39)
3       2026-11-24 13:47:38 +0100 CET  HEAD (8620509)
4       2026-11-24 13:51:26 +0100 CET  HEAD (733ccb4)
5       2026-11-24 13:52:57 +0100 CET  HEAD (f2710bd)
6       2026-11-24 13:55:21 +0100 CET  HEAD (21527cf)
7       2026-11-24 14:00:27 +0100 CET  HEAD (c0adc2d)
8       2026-11-24 14:06:06 +0100 CET  HEAD (c7dd7ba)
```

- Sortie de `argocd app list` :

```
NAME              CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                          PATH                            TARGET
argocd/concierge  https://kubernetes.default.svc  staybook   default  Synced  Healthy  Auto-Prune  <none>      https://github.com/Ikdek/staybook-gitops.git  staybook-gitops/apps/concierge  HEAD
argocd/staybook   https://kubernetes.default.svc  staybook   default  Synced  Healthy  Auto-Prune  <none>      https://github.com/Ikdek/staybook-gitops.git  staybook-gitops/apps/staybook   HEAD
```
