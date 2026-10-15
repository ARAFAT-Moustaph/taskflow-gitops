# Mastère DevOps M2 — Sup de Vinci
## Module E54 : CI/CD Avancé & Déploiement Continu
### Bloc 5, RNCP 40165 · Activité 12b : GitOps et Stratégies de Release

---

## Composition de l'Équipe
- **Aimen KLAI** ([@aimenklai](https://github.com/aimenklai))
- **Moustapha ARAFAT** ([@ARAFAT-Moustaph](https://github.com/ARAFAT-Moustaph))

**Dépôt GitOps officiel :** [https://github.com/ARAFAT-Moustaph/taskflow-gitops](https://github.com/ARAFAT-Moustaph/taskflow-gitops)

---

## 1. Principes d'Architecture & Boucle GitOps

Le principe fondamental appliqué tout au long de ce projet est que **Git constitue l'unique source de vérité (*Single Source of Truth*) pour l'état du cluster**. 

Toute modification en production est initiée sous la forme d'un commit tracé, soumis à revue par les pairs via une Pull Request sur GitHub protégée par un Ruleset. L'agent interne **Argo CD** surveille la branche principale et synchronise l'état réel de Kubernetes en mode Pull.

### Réponses aux questions de la Slide 9

#### Question 1 : Déploiement Push ou Pull ?
Notre architecture fonctionne selon le modèle **Pull (GitOps)** :
- **Modèle Push :** Le pipeline CI externe dispose d'accès privilégiés (`kubeconfig` administrateur) et pousse directement les manifests sur l'API Kubernetes. Cela expose des secrets critiques en dehors du cluster et ne permet pas de réagir aux modifications directes faites en local.
- **Modèle Pull :** L'agent Argo CD s'exécute à l'intérieur du cluster. Il interroge périodiquement le dépôt Git (toutes les 60 secondes) et réconcilie l'état réel avec l'état déclaré. Aucun port d'administration n'est exposé vers l'extérieur et la moindre dérive non déclarée dans Git est automatiquement neutralisée.

#### Question 2 : Qui a corrigé quoi lors de la dérive (Drift) ?
Lors de l'altération manuelle du cluster via :
```bash
kubectl scale deployment taskflow -n taskflow --replicas=8
```
C'est **Argo CD** qui a détecté la divergence grâce à sa configuration `selfHeal: true`. Argo CD a comparé les 8 pods actifs avec la valeur déclarée dans Git (`replicas: 4`). En **9 secondes**, il a passé les 4 pods surnuméraires en statut `Terminating`, rétablissant strictement l'état consigné dans Git.

#### Question 3 : Pourquoi faire un `git revert` plutôt qu'une commande manuelle sur le cluster ?
Une commande impérative telle que `kubectl rollout undo` ou `kubectl set image` est immédiatement écrasée par la réconciliation automatique d'Argo CD.
Faire un `git revert` présente trois avantages majeurs :
1. **Intégrité de la source de vérité :** Le cluster reste strictement conforme à la branche `main`.
2. **Auditabilité et traçabilité :** L'annulation génère un commit d'historique explicite et passe par le processus standard de Pull Request avec validation par un pair.
3. **Réversibilité :** L'historique n'est pas altéré, ce qui permet d'auditer ou de ré-appliquer les changements si nécessaire.

---

## 2. Analyse Technique du Déploiement Blue-Green (Partie A)

### Pourquoi avoir mis en place le Blue-Green ?
Le contrôleur par défaut de Kubernetes (`Deployment`) utilise une stratégie de *Rolling Update*. Durant la transition, les anciennes et les nouvelles versions cohabitent sur le même service et se partagent les requêtes des utilisateurs. Si la nouvelle version comporte une régression critique, une partie du trafic utilisateur subit immédiatement des erreurs.

La stratégie **Blue-Green** résout ce problème en séparant strictement l'environnement de production de l'environnement de qualification :
- **Version active (Bleu) :** Les pods en version stable (1.0.0) reçoivent 100% du trafic via le service `taskflow`.
- **Version candidate (Vert) :** Les pods en nouvelle version (1.1.0) démarrent sur un service distinct et isolé, `taskflow-preview`. Les tests de validation s'effectuent sans exposer le moindre utilisateur de production.

### Pourquoi le nombre de pods a doublé (4 pods -> 8 pods) ?
Durant la phase de prévisualisation, nous avons constaté la présence simultanée de **8 pods** dans le namespace `taskflow` :
- 4 pods hébergeant la version 1.0.0 (ReplicaSet actif).
- 4 pods hébergeant la version 1.1.0 (ReplicaSet preview).

Ce doublement est obligatoire dans une architecture Blue-Green : pour que la prévisualisation soit représentative de la charge de production, l'environnement candidat doit être instancié à pleine capacité avant la bascule.

**Conséquence sur les ressources :**
- Empreinte CPU requise : `8 x 50m` = **400m de CPU**.
- Empreinte mémoire requise : `8 x 64Mi` = **512 Mi de RAM**.
Le Blue-Green implique donc une surallocation temporaire de **100% des ressources** matérielles pendant toute la durée des tests de validation.

### Bascule (Promotion) et drainage gracieux
Une fois la conformité validée sur `taskflow-preview` via le script `observe.sh` (100% de réponses HTTP 200 en version 1.1.0), la bascule est déclenchée par la commande :
```bash
kubectl argo rollouts promote taskflow -n taskflow
```
- **Mécanisme :** Le contrôleur Argo Rollouts modifie instantanément le sélecteur d'étiquettes du service de production `taskflow` pour pointer vers le ReplicaSet vert. Le basculement est immédiat (délai réseau quasi nul).
- **Rôle du paramètre `scaleDownDelaySeconds: 30` :** Les 4 anciens pods bleus ne sont pas détruits instantanément. Un délai de 30 secondes est observé pour permettre le drainage gracieux des requêtes HTTP en cours sur l'ancienne version, évitant toute rupture brutale de session pour les clients connectés.

---

## 3. Analyse Technique du Déploiement Canary (Partie B)

### Pourquoi passer au Canary ?
Bien que le Blue-Green apporte une isolation totale, il présente deux limites majeures en entreprise :
1. **Le coût d'infrastructure :** Doubler le nombre de pods n'est pas viable sur des architectures volumineuses (ex: passer de 200 à 400 pods génère un surcoût Cloud important).
2. **L'effet tout-ou-rien :** La promotion bascule l'intégralité du trafic d'un coup. Un bug latent qui ne se manifeste que sous forte charge réelle impacte alors 100% des utilisateurs.

La stratégie **Canary** pallie ces défauts en acheminant progressivement une fraction du trafic réel vers la nouvelle version sans surallocation massive de ressources.

### Répartition statistique du trafic (25% / 75%)
Sans maillage de services (*Service Mesh* type Istio ou Linkerd), le routage du trafic repose sur la proportion de pods dans le Service Kubernetes.
Avec `replicas: 4` et une première étape `setWeight: 25` :
- 1 pod exécute la version candidate 2.0.0.
- 3 pods exécutent la version stable 1.1.0.
- Total : **4 pods maintenus**, aucune surconsommation de ressources.

L'exécution de `./scripts/observe.sh taskflow 40` a confirmé cette répartition statistique réelle :
- **27 requêtes** traitées par la version 1.1.0 (~70%).
- **13 requêtes** traitées par la version 2.0.0 (~30%).

Le rollout s'est mis en pause automatique (`CanaryPauseStep`), permettant d'observer la stabilité de la version candidate sur un échantillon d'utilisateurs.

### Test de défaillance applicative (v2.1.0) et coupure d'urgence (`abort`)
Pour valider la résilience du modèle, nous avons déployé l'image `2.1.0` (PR #11).
L'exécution de `observe.sh` a mis en évidence des pannes applicatives immédiates :
```text
33 version=2.0.0 http=200
 5 version=2.1.0 http=200
 2 version=aucune http=500
```
Pour couper l'incident et protéger les utilisateurs, la commande d'urgence a été exécutée :
```bash
kubectl argo rollouts abort taskflow -n taskflow
```
Le contrôleur a instantanément retiré le pod canari 2.1.0 du trafic et redirigé l'ensemble des flux vers la version 2.0.0 saine (`40 version=2.0.0 http=200`).

### Réponse à la question : « ... et après ? »
La commande `abort` isole l'incident au niveau du cluster, mais dans Git, la branche `main` pointe toujours vers le commit introduisant la version 2.1.0 défaillante.
Si Argo CD effectue un nouveau cycle de synchronisation forcée, il tentera de réappliquer l'état déclaré dans Git.
Pour réaligner la source de vérité avec le cluster, il est impératif d'effectuer un **`git revert` sur GitHub** (PR #12). Ce processus garantit que la version stable 2.0.0 redevient la référence déclarée dans Git.

---

## 4. Comparatif : « Blue-Green ou Canary pour TaskFlow ? »

| Critère d'évaluation | Stratégie Blue-Green | Stratégie Canary |
|---|---|---|
| **Exposition au risque** | **Nulle (0%)** : La nouvelle version est testée sur un service isolé avant la moindre mise en production. | **Limitée mais réelle** : 25% des vrais utilisateurs essuient les plâtres de la nouvelle version. |
| **Consommation de ressources (Coût)** | **Élevée (+100%)** : Nécessite 8 pods simultanés pour faire tourner les deux environnements en parallèle. | **Optimale (0% de surcoût)** : Maintien strict des 4 pods alloués en remplaçant les réplicas un à un. |
| **Vitesse de bascule** | **Instantanée** : Simple modification de l'aiguillage du Service Kubernetes. | **Progressive** : Passage par paliers configurés (25% -> 50% -> 75% -> 100%). |
| **Vitesse de rollback** | **Immédiate** : Rétablissement du pointeur de service vers l'ancien ReplicaSet. | **Immédiate** via `argo rollouts abort`, suivie d'un `git revert` obligatoire. |
| **Gestion des bases de données** | Délicate : impose une rétrocompatibilité stricte des schémas de données pour deux versions actives. | Délicate également si la version candidate altère la base. |

### Conclusion et recommandation pour TaskFlow
Dans le contexte actuel de **TaskFlow** (application légère dimensionnée à 4 réplicas consommant peu de mémoire) :
- La stratégie **Blue-Green est recommandée**. Le coût matériel de 4 pods supplémentaires est négligeable en regard du bénéfice d'isolation totale : aucun client n'est exposé à une version non testée.
- Si l'application venait à monter en charge (ex: plusieurs centaines de pods distribués sur plusieurs nœuds), le surcoût de doublement d'infrastructure deviendrait prohibitif. Dans ce scénario de mise à l'échelle, la stratégie **Canary** s'imposerait pour des impératifs économiques et de contrôle granulaire du trafic.

---

## 5. Journal des Déploiements

| Heure | Version | Étape / Action | Acteur / Mécanisme | Délai mesuré | Résultat technique observé |
|---|---|---|---|---|---|
| **12:12** | `1.0.0` | Déploiement initial | `kubectl apply -f argocd/application.yaml` | Immédiat | Application initialisée dans Argo CD. 4 pods opérationnels. 40/40 requêtes en 1.0.0 (HTTP 200). |
| **14:52** | `2.0.0` | Montée de version majeure | PR #1 mergée sur `main` | ~40 s | Rolling Update automatique déclenché par Argo CD. Bascule validée à 40/40 en 2.0.0 (HTTP 200). |
| **14:56** | `2.0.0` | Test de dérive (*Drift*) | `kubectl scale --replicas=8` manuel | 9 s | Détection Argo CD : 4 pods pirates passés en `Terminating`. Rétablissement strict des 4 pods de Git. |
| **15:00** | `1.0.0` | Rollback d'urgence | PR #2 (`git revert`) mergée sur `main` | ~35 s | Rétrogradation automatique par Argo CD. 40/40 requêtes en 1.0.0 (HTTP 200). |
| **16:33** | `1.0.0` | Migration vers Rollout Blue-Green | PR #6 mergée sur `main` | ~30 s | Remplacement du `Deployment` par l'objet `Rollout` et création de `taskflow-preview`. Application saine. |
| **16:38** | `1.1.0` | Déploiement Blue-Green | PR #7 mergée sur `main` | ~45 s | Dédoublement constaté : 4 pods v1.0.0 (prod) + 4 pods v1.1.0 (preview) = 8 pods au total. Promotion manuelle et extinction des anciens pods après 30s. |
| **16:57** | `1.1.0` | Migration vers Rollout Canary | PR #8 mergée sur `main` | ~25 s | Suppression du service preview et bascule du contrôleur Rollout en stratégie Canary. |
| **17:00** | `2.0.0` | Déploiement Canary progressif | PR #10 mergée sur `main` | ~20 s | Rollout en pause à 25% de trafic : `observe.sh` mesure 27 requêtes en 1.1.0 et 13 requêtes en 2.0.0 (HTTP 200). Promotion à 100% validée. |
| **17:11** | `2.1.0` | Injection de défaillance applicative | PR #11 mergée sur `main` | ~15 s | Rollout en pause à 25% : `observe.sh` détecte des erreurs `http=500` sur la version candidate 2.1.0. |
| **17:13** | `2.0.0` | Arrêt d'urgence de la défaillance | `kubectl argo rollouts abort` | Immédiat | Retrait immédiat du pod canari défaillant. Rétablissement à 100% du trafic sain en version 2.0.0. |
| **17:16** | `2.0.0` | Réalignement de la source Git | PR #12 (`git revert`) mergée sur `main` | ~30 s | Réconciliation GitOps définitive : Git et cluster alignés sur la version stable 2.0.0. |

---

## 6. Galerie des Preuves Visuelles

### Preuve 1 : Protection de la branche principale (Ruleset GitHub)
Configuration imposant la Pull Request obligatoire, l'approbation minimale par un pair et l'interdiction des pushs forcés sur `main`.
![Preuve 1 - Ruleset](screenshots/01-ruleset.png)

### Preuve 2 : Topologie complète dans l'interface Argo CD
Visualisation de l'application `taskflow` sous Argo CD montrant les statuts **Synced** et **Healthy** et l'arbre des pods opérationnels.
![Preuve 2 - Argo CD UI](screenshots/02-argocd-ui.png)

### Preuve 3 : Pull Request #1 de montée en v2.0.0 (Approbation)
Démonstration de la revue par les pairs : Pull Request créée par Aimen KLAI et approuvée par Moustapha ARAFAT avant merge.
![Preuve 3 - PR v2 Approved](screenshots/03-pr1-approved.png)

### Preuve 4 : Déploiement effectif de la version 2.0.0
Exécution du script `./scripts/observe.sh taskflow 40` démontrant que 100% du trafic reçoit une réponse HTTP 200 de la version 2.0.0.
![Preuve 4 - Observe v2](screenshots/04-observe-v2.png)

### Preuve 5 : Détection de dérive et auto-guérison (Self-Healing)
Tentative d'altération manuelle (`scale 8`) et réaction d'Argo CD qui passe les pods hors-Git en `Terminating` à 9 secondes pour revenir à 4 réplicas.
![Preuve 5 - Drift & Self-Healing](screenshots/05-drift-selfheal.png)

### Preuve 6 : Rollback propre par Git Revert (Pull Request #2)
Pull Request de revert validée et fusionnée, ramenant le cluster à la version 1.0.0 de façon tracée.
![Preuve 6 - Git Revert PR](screenshots/06-pr2-revert.png)

### Preuve 7 : PR #6 de migration vers le Rollout Blue-Green
Pull Request validée remplaçant le `Deployment` par un contrôleur `Rollout` Argo avec stratégie Blue-Green et service de prévisualisation.
![Preuve 7 - PR BlueGreen Rollout](screenshots/07-bg-pr-rollout.png)

### Preuve 8 : PR #7 de montée de version Blue-Green en 1.1.0
Approbation et fusion de la PR déclenchant le déploiement de la version 1.1.0 sur le service isolé `taskflow-preview`.
![Preuve 8 - PR BlueGreen 1.1.0](screenshots/08-bg-pr-v110.png)

### Preuve 9 : PR #8 de migration vers l'architecture Canary
Suppression du service de prévisualisation et configuration des étapes progressives de trafic dans le Rollout.
![Preuve 9 - PR Canary Rollout](screenshots/09-canary-pr-rollout.png)

### Preuve 10 : PR #10 de déploiement progressif en 2.0.0
Pull Request initiant le déploiement canari de la version 2.0.0.
![Preuve 10 - PR Canary v2](screenshots/10-canary-pr-v2.png)

### Preuve 11 : Répartition statistique réelle du trafic à 25% (Canary)
Capture du terminal `./scripts/observe.sh taskflow 40` démontrant la cohabitation contrôlée : 27 requêtes traitées en 1.1.0 et 13 requêtes traitées en 2.0.0 sans interruption de service.
![Preuve 11 - Canary 25%](screenshots/11-canary-traffic-25.png)

### Preuve 12 : Promotion définitive à 100% de la version 2.0.0
Exécution de `observe.sh` confirmant le basculement complet de la production : 40/40 requêtes en 2.0.0 (HTTP 200).
![Preuve 12 - Canary 100%](screenshots/12-canary-traffic-100.png)

### Preuve 13 : PR #11 d'injection de défaillance applicative (2.1.0)
Déploiement de la version candidate 2.1.0 pour évaluer la détection d'anomalies.
![Preuve 13 - PR v2.1.0](screenshots/13-canary-pr-v210-error.png)

### Preuve 14 : Constat des erreurs HTTP 500 sur le Canari
Observation en direct de la dégradation applicative via `./scripts/observe.sh` (codes d'erreur `http=500` isolés sur la version candidate).
![Preuve 14 - Erreur HTTP 500](screenshots/14-canary-http-500.png)

### Preuve 15 : PR #12 de Revert Git suite à l'incident
Annulation tracée de la version défaillante sur GitHub, réalignant l'état déclaré dans Git avec le cluster sécurisé.
![Preuve 15 - Revert Git Incident](screenshots/15-canary-revert-pr.png)

---

## 7. Guide de Reproduction des Commandes Utilisées

### 1. Surveillance et observation du trafic
```bash
# Vérifier la répartition du trafic et les codes HTTP
./scripts/observe.sh taskflow 40

# Vérifier le service de prévisualisation (Blue-Green)
./scripts/observe.sh taskflow-preview 20
```

### 2. Pilotage d'Argo Rollouts en ligne de commande
```bash
# Consulter l'état en direct du Rollout
~/.local/bin/kubectl-argo-rollouts get rollout taskflow -n taskflow

# Suivre en temps réel les changements d'étapes
~/.local/bin/kubectl-argo-rollouts get rollout taskflow -n taskflow --watch

# Promouvoir le déploiement vers l'étape suivante (ou 100%)
~/.local/bin/kubectl-argo-rollouts promote taskflow -n taskflow

# Déclencher un arrêt d'urgence immédiat en cas d'anomalie
~/.local/bin/kubectl-argo-rollouts abort taskflow -n taskflow
```

### 3. Contrôle des pods Kubernetes
```bash

---

## 8. Lab Robustesse & Analyse Automatique (Canary 2.0.0 → 2.1.0 → Abort Automatique → 2.2.0)

### 8.1 Objectif et rupture avec le déploiement manuel
Dans le TP précédent, l'interruption d'une version bugguée nécessitait une surveillance visuelle humaine et une action manuelle en CLI (`kubectl argo rollouts abort`).  
Ce lab met en œuvre le standard de l'ingénierie **SRE (Site Reliability Engineering)** : l'**Automated Canary Analysis (ACA)**. C'est le contrôleur **Argo Rollouts** qui lance de manière autonome un test de charge (**k6**) dès le premier palier de trafic. Si les métriques ne respectent pas les seuils de qualité fixés, le déploiement est **avorté instantanément de manière automatique**, sans aucune action humaine.

Le rapport d'incident complet est consigné dans le document officiel : [docs/postmortem-2.1.0.md](docs/postmortem-2.1.0.md).

---

### 8.2 Phase A : Mesure de l'Étalon de référence (*Baseline*)
Avant d'injecter des anomalies, nous mesurons les performances de référence sur la version stable en production (**v2.0.0**) via `./scripts/charge.sh http://taskflow` (5 VUs, 30 secondes).

| Métrique | Seuil d'acceptabilité | Valeur mesurée (v2.0.0) | Statut |
| :--- | :--- | :--- | :--- |
| **Taux d'erreur (`http_req_failed`)** | `< 2.00 %` | **`0.00 %`** (0 échec / 708 req) | ✅ Validé |
| **Latence p95 (`http_req_duration`)** | `< 250 ms` | **`21.71 ms`** | ✅ Validé |
| **Statut global** | Code sortie 0 | 100 % de codes HTTP 200 | ✅ Étalon conforme |

---

### 8.3 Phase B : L'Incident 2.1.0 et l'Interception Automatique

#### 1. Déploiement de la version 2.1.0
* **Pull Request #15** mergée sur `main` avec la version `2.1.0`.
* Argo Rollouts alloue 25 % du trafic (1 pod révision 5) et attend 20 secondes (`CanaryPauseStep`).
* Lancement automatique de l'objet **`AnalysisRun`** (`taskflow-df976ccb5-5-2`) qui déclenche un Job k6 ciblant `http://taskflow-canary`.

#### 2. Les 4 preuves de l'enquête technique
1. **Preuve n°1 (Statut Rollout)** : Le Rollout bascule en **`✖ Degraded`** avec le message `Metric "test-de-charge-k6" assessed Failed due to failed (1) > failureLimit (0)`. Le pod défaillant est immédiatement stoppé (`ScaledDown`).
2. **Preuve n°2 (Statut AnalysisRun)** : `taskflow-df976ccb5-5-2` passe en **`Failed`**.
3. **Preuve n°3 (Logs du test k6)** : Le test k6 a mesuré **28.81 % d'erreurs HTTP** (85 requêtes échouées sur 295) et une latence **p95 de 327.1 ms** (violant le seuil de 250 ms).
4. **Preuve n°4 (Cécité des probes Kubernetes)** : Les sondes `readinessProbe` et `livenessProbe` interrogeaient uniquement `/health` (qui renvoyait un faux positif `200 OK`). Le bug impactait exclusivement l'endpoint métier `/tasks` (erreurs `HTTP 500`). Seul le test k6 a pu dévoiler la défaillance.

#### 3. Résolution du Drift Git
* Le cluster a réagi seul, mais la branche `main` pointait toujours vers `2.1.0`.
* Pour réaligner Git avec la production, la **Pull Request #16** de Git Revert a été soumise, approuvée et mergée.

---

### 8.4 Phase C : Déploiement de la version corrective 2.2.0
1. Mise à jour de `rollout.yaml` avec l'image `ghcr.io/9m7fjfpv9k-cyber/taskflow:2.2.0`.
2. Soumission et merge de la **Pull Request #17**.
3. Au palier 25 %, l'AnalysisRun `taskflow-7ddd57d788-7-2` s'exécute avec succès :
   * **Taux d'erreur** : `0.00 %`
   * **Latence p95** : `< 250 ms`
   * **Statut AnalysisRun** : **`✔ Successful`**
4. Argo Rollouts déclenche l'avancement automatique : passage à 50 % (pause 30s) → 75 % (pause 30s) → **100 %**.
5. Statut final du cluster : **`✔ Healthy`** avec les 4 pods en version 2.2.0.

---

### 8.5 Galerie de Preuves du Lab Robustesse

| Preuve | Description | Capture |
| :--- | :--- | :--- |
| **Preuve 16** | **Étalon de référence v2.0.0** : Sortie terminal k6 validant p95=21.71ms et 0 % d'erreurs | ![Étalon k6 v2.0.0](screenshots/16-etalon-k6-2.0.0.png) |
| **Preuve 17** | **PR #14** : Ajout de la configuration d'analyse automatique k6 (4 fichiers) | ![PR 14 Analyse Auto](screenshots/17-pr14-analyse-auto.png) |
| **Preuve 18** | **Ressources K8s actives** : Déploiement de l'AnalysisTemplate, ConfigMap et Services | ![Ressources CRD](screenshots/18-k8s-crd-resources.png) |
| **Preuve 19** | **Zéro Deployment** : Vérification de l'absence totale de Deployments concurrents | ![Zéro Deployment](screenshots/19-zero-deployment.png) |
| **Preuve 20** | **PR #15** : Déploiement de la version candidate 2.1.0 | ![PR 15 Canary 2.1.0](screenshots/20-pr15-canary-2.1.0.png) |
| **Preuve 21** | **Abort Automatique v2.1.0** : Rollout basculé en Degraded sans action humaine | ![AnalysisRun Échec 2.1.0](screenshots/21-analysisrun-echec-2.1.0.png) |
| **Preuve 22** | **Preuve 1 Enquête** : AnalysisRun marqué `Failed` dans Kubernetes | ![Preuve 1 AnalysisRun Failed](screenshots/22-proof1-analysisrun-failed.png) |
| **Preuve 23** | **Preuve 2 Enquête** : Logs k6 démontrant 28.81 % d'erreurs HTTP | ![Preuve 2 k6 logs failed](screenshots/23-proof2-k6-logs-failed.png) |
| **Preuve 24** | **PR #16 de Revert** : Annulation GitOps de la PR 2.1.0 sur GitHub | ![Revert PR 16](screenshots/24-revert-pr16-canary-2.1.0.png) |
| **Preuve 25** | **Rollout 2.2.0 Healthy** : Basculement à 100 % sur la version corrigée | ![Rollout Healthy 2.2.0](screenshots/25-rollout-healthy-2.2.0.png) |
| **Preuve 26** | **AnalysisRun en Succès et Échec** : Vue comparative des deux analyses côte à côte | ![AnalysisRun Succès 2.2.0](screenshots/26-analysisrun-succes-2.2.0.png) |
