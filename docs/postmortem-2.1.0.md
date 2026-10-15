# Postmortem — Défaillance applicative de la version 2.1.0 et Rollback automatique par Canary Analysis

> **Principe de culture SRE** : Sans reproche (*blameless postmortem*). On cherche ce qui a permis l'erreur et comment le système a réagi pour protéger la production, pas qui l'a commise.

| Champ | Valeur |
| :--- | :--- |
| **Date et heure** | 15 octobre 2026, 11:58:12 — 11:59:18 CEST (UTC+2) |
| **Version en cause** | `ghcr.io/9m7fjfpv9k-cyber/taskflow:2.1.0` |
| **PR à l'origine** | [Pull Request #15](https://github.com/ARAFAT-Moustaph/taskflow-gitops/pull/15) (`feat: deploiement version 2.1.0 avec analyse automatique`) |
| **Durée d'exposition** | 66 secondes (du démarrage du pod canary à 11:58:12 jusqu'au scaling down automatique à 11:59:18) |
| **Part du trafic touché** | 0 % du trafic externe de production. 25 % du trafic interne isolé sur le service dédié `taskflow-canary` via le palier de test k6 |
| **Détecté par** | Test de charge k6 automatique orchestré par l'objet CRD `AnalysisRun` d'Argo Rollouts |
| **Résolu par** | Avortement automatique du Rollout (`RolloutAborted`) en 2 secondes après l'échec de k6, suivi d'un Git Revert ([PR #16](https://github.com/ARAFAT-Moustaph/taskflow-gitops/pull/16)) |

---

## 1. Chronologie détaillée des événements

| Horodatage (UTC) | Événement | Composant / Acteur |
| :--- | :--- | :--- |
| **09:57:34** | Merge de la PR #15 introduisant l'image `2.1.0` sur la branche `main` | GitOps / GitHub |
| **09:58:05** | Argo CD détecte le commit, passe en `Progressing` et applique la révision 5 sur Kubernetes | Argo CD Application Controller |
| **09:58:12** | Step 1/7 : Rollout active le palier 25 % (`setWeight: 25`). Scaled up ReplicaSet `taskflow-df976ccb5` de 0 à 1 pod (`taskflow-df976ccb5-6zhhf`) | Argo Rollouts Controller |
| **09:58:14** | Activation de la pause de 20 secondes (`pause: 20s`) pour stabiliser le pod et laisser le temps au service canary de basculer | Argo Rollouts |
| **09:58:33** | Step 2/7 : Création de l'AnalysisRun `taskflow-df976ccb5-5-2` et lancement du Job k6 `1186d9ce-...` | Argo Rollouts / Batch API |
| **09:58:34** | k6 démarre l'envoi de requêtes HTTP sur `http://taskflow-canary/tasks` (5 VUs, durée 30s) | Conteneur `grafana/k6:latest` |
| **09:59:06** | k6 termine le tir et constate des erreurs massives HTTP 500. Seuil `http_req_failed` violé (28.81 % > 2 %). Le Job se termine avec code 1 (`Error`) | Pod k6 |
| **09:59:16** | La métrique `test-de-charge-k6` est évaluée en échec (`failed (1) > failureLimit (0)`). L'AnalysisRun passe en `✖ Failed` | Argo Rollouts |
| **09:59:18** | **Déclenchement de l'Abort Automatique** : Rollout passe en `✖ Degraded`. Le service `taskflow-canary` est immédiatement rebasculé sur l'ancienne révision `c6cf57bd6`. Le pod `2.1.0` est détruit (`Terminating`), et le ReplicaSet stable `2.0.0` repasse à 4 pods complets | Argo Rollouts |
| **10:17:45** | Ouverture et merge de la [PR #16](https://github.com/ARAFAT-Moustaph/taskflow-gitops/pull/16) (Git Revert) pour éliminer la dérive (*drift*) entre Git et le cluster | Équipe (Aimen & Moustapha) |
| **10:25:39** | Merge de la [PR #17](https://github.com/ARAFAT-Moustaph/taskflow-gitops/pull/17) déployant la version corrigée `2.2.0` | Équipe (Aimen & Moustapha) |
| **10:27:57** | L'AnalysisRun `taskflow-7ddd57d788-7-2` réussit (`✔ Successful`). Progression automatique sans interruption jusqu'à 100 % (`✔ Healthy`) | Argo Rollouts |

---

## 2. Composant défaillant et cause racine

### A. Quel composant a échoué ?
Le composant applicatif contenu dans l'image conteneur **`ghcr.io/9m7fjfpv9k-cyber/taskflow:2.1.0`**.  
Lors des appels sur la route métier `/tasks`, l'application a renvoyé des erreurs HTTP 500 en rafale.

**Preuve d'exécution (logs du Job k6 `pod/1186d9ce-d258-444c-b4c4-007f9da419e1.test-de-charge-k6.1-cfk74`) :**
```text
  █ THRESHOLDS 

    http_req_duration
    ✗ 'p(95)<250' p(95)=327.1ms

    http_req_failed
    ✗ 'rate<0.02' rate=28.81%

  █ TOTAL RESULTS 

    checks_total.......: 295    9.487874/s
    checks_succeeded...: 71.18% 210 out of 295
    checks_failed......: 28.81% 85 out of 295

    ✗ statut 200
      ↳  71% — ✓ 210 / ✗ 85

    HTTP
    http_req_duration..............: avg=311.91ms min=301.68ms med=308.34ms max=464.91ms p(90)=321.34ms p(95)=327.1ms
    http_req_failed................: 28.81% 85 out of 295
```

### B. Pourquoi les sondes Kubernetes (*Probes*) ne l'ont-elles pas vu ?
Dans la configuration du Rollout ([rollout.yaml](file:///d:/M2_PROJET_2026_2027/TP_GITLAB_CI/GitOps/taskflow-gitops/apps/taskflow/rollout.yaml)), les sondes Kubernetes de disponibilité et de vivacité sont configurées ainsi :

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: http
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /health
    port: http
  periodSeconds: 10
```

* **Le piège technique** : Le serveur web HTTP démarré dans le conteneur répondait systématiquement `200 OK` sur la route de santé technique `/health` (`{"status":"ok"}`).
* Pour Kubernetes kubelet, le conteneur était donc considéré à 100 % opérationnel (`ready: 1/1`).
* En revanche, les requêtes des utilisateurs ciblant la route applicative `/tasks` échouaient à 28.81 %.
* **Conclusion** : Une simple sonde technique superficielle est insuffisante pour détecter une régression métier sous charge. Seule une analyse canary automatisée avec injection de trafic de test (`AnalysisTemplate` + k6) a pu dévoiler la défaillance.

### C. Cause racine (*Root Cause*)
Régression logicielle interne à la version `2.1.0` affectant le traitement des collections de tâches, combinée à une absence de tests d'intégration fonctionnels préalables au packaging de l'image.

---

## 3. Ce qui a bien fonctionné

1. **Isolation totale du trafic canary** :  
   L'utilisation de la directive `canaryService: taskflow-canary` a permis d'orienter le tir de charge k6 uniquement vers les pods de la révision 5 sans polluer le service de production `taskflow` des utilisateurs.
2. **Détection précoce dès le palier 25 %** :  
   Le bug a été intercepté alors qu'un seul pod sur 4 était déployé, évitant un déploiement complet en production.
3. **Réaction instantanée sans intervention humaine** :  
   Dès que le Job k6 a échoué, Argo Rollouts a mis **moins de 2 secondes** pour basculer le Rollout en `✖ Degraded`, tuer le pod défaillant et réallouer 100 % des 4 réplicas à l'ancienne révision stable `2.0.0`.
4. **Gestion de crise conforme GitOps** :  
   L'incident a été résolu sur le cluster par le contrôleur Kubernetes, puis régularisé dans le dépôt Git par une Pull Request de Revert ([PR #16](https://github.com/ARAFAT-Moustaph/taskflow-gitops/pull/16)) validée en binôme.

---

## 4. Actions correctives et préventives

| Action | Type | Responsable | Échéance | Statut |
| :--- | :--- | :--- | :--- | :--- |
| **Déployer la version corrective v2.2.0** | Correctif immédiat | Aimen KLAI | J2 | ✅ Réalisé ([PR #17](https://github.com/ARAFAT-Moustaph/taskflow-gitops/pull/17)) |
| **Maintenir l'analyse de robustesse k6 automatique** | Robustesse | Moustapha ARAFAT | J2 | ✅ Intégré dans `apps/taskflow` |
| **Améliorer la `readinessProbe`** pour vérifier la connectivité de la base et la disponibilité de `/tasks` plutôt qu'un `/health` statique | Préventif | Binôme Dev/Ops | Sprint suivant | 📋 Planifié |
| **Ajouter un seuil d'échec strict dans la CI avant la génération de tag Docker** pour valider les endpoints critiques | Qualité CI | Binôme Dev/Ops | Sprint suivant | 📋 Planifié |
