# RAPPORT — TP 3 GitOps : DevHub Campus SRE

## Binôme
- Valentin (valaack)

## Outillage TP 3 (Étape 0)
- kubectl-argo-rollouts : v1.9.0
- promtool : 2.55.1
- jq : 1.7

## Étape 1 — SLI, SLO, error budget

### Tableau par service

| Service | SLI | SLO | Error budget mensuel |
|---|---|---|---|
| annuaire | Disponibilité | 99,5 % | 3h36 |
| annuaire | Latence p95 | < 300 ms | 3h36/mois sous le seuil |
| annuaire | Taux d'erreur 5xx | < 1 % | -- |
| planning | Disponibilité | 99,5 % | 3h36 |
| planning | Latence p95 | < 500 ms (calculs de creneaux, plus lourd) | 3h36/mois |
| planning | Taux d'erreur 5xx | < 1 % | -- |
| notif | Disponibilité | 99 % | 7h12 |
| notif | Latence p95 | < 200 ms (service leger Go) | 7h12/mois |
| notif | Taux d'erreur 5xx | < 2 % (moins critique) | -- |

### PromQL pseudo-code

Disponibilite (annuaire) :
sum(rate(http_requests_total{service="annuaire", status_class!~"5.."}[5m])) / sum(rate(http_requests_total{service="annuaire"}[5m]))

Latence p95 (annuaire) :
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service="annuaire"}[5m])) by (le))

Taux d'erreur 5xx (annuaire) :
sum(rate(http_requests_total{service="annuaire", status_class="5xx"}[5m])) / sum(rate(http_requests_total{service="annuaire"}[5m]))

### Prise de position

Pour annuaire, l'error budget est de 3h36 par mois. Si on l'epuise en deux semaines, je regarde d'abord les deploiements recents (un canary qui aurait du etre bloque), puis je gele les deploiements non critiques jusqu'a la fin du mois, et j'ouvre un post-mortem pour comprendre pourquoi l'AnalysisTemplate n'a pas intercepte le probleme en amont.

J'ai choisi 99,5 % plutot que 99,99 % car un SLO trop strict qu'on ne tient jamais rend l'error budget toujours negatif -- autant ne pas en avoir. 99,5 % reste ambitieux mais atteignable pour un service interne comme annuaire.

## Étape 2 — Configuration des buckets (annuaire)

Les buckets par defaut du chart sont 0.05, 0.1, 0.2, 0.3, 0.5, 1, 2, 5 secondes.

Pour annuaire, ces buckets sont conserves tels quels car mon SLO p95 est de 300 ms : le bucket 0.3 est present exactement, avec des points encadrants (0.1, 0.2, 0.5) qui garantissent un quantile p95 precis a ce niveau.

Validation :
- curl /metrics retourne du Prometheus valide
- promtool check metrics ne signale que 3 warnings de convention sur des metriques par defaut de prom-client (nodejs_active_handles_total etc), sans impact sur les metriques RED ecrites pour le TP

## Étape 4 — ServiceMonitor + dashboard Grafana (annuaire)

Le label `release: kps` est ajoute au Service annuaire via le flag `monitoring.enabled`, ce qui permet au ServiceMonitor du kube-prometheus-stack de le decouvrir automatiquement (selecteur `matchLabels` sur les labels du Service annuaire).

Dashboard "annuaire - DevHub Campus" avec 4 panneaux :

1. RPS par route -- sum(rate(http_requests_total{namespace="devhub-dev"}[5m])) by (route)
   Montre le volume de trafic par endpoint, utile pour detecter un pic ou une chute anormale.

2. Taux d'erreur 5xx -- sum(rate(http_requests_total{status_class="5xx"}[5m])) / sum(rate(http_requests_total[5m]))
   Ratio d'erreurs serveur sur le total des requetes, indicateur direct du SLO de fiabilite.

3. Latence p50/p95/p99 -- histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
   Compare les percentiles de latence pour voir si la majorite des requetes est rapide tout en surveillant la queue (p99).

4. Build info -- annuaire_build_info{namespace="devhub-dev"}
   Affiche le commit et la version deployee, utile pour confirmer qu'un rollout a bien pousse la bonne image.

Dashboard exporte en JSON et commite dans platform-sre/dashboards/annuaire.json.

## Etape 5 -- Du Deployment au Rollout (canary sur annuaire)

Migration du chart annuaire : suppression de deployment.yaml et service.yaml, ajout de rollout.yaml (kind: Rollout, strategy.canary), service-stable.yaml et service-canary.yaml.

Strategie : setWeight 20, pause 30s, setWeight 50, pause 30s, setWeight 100. Traffic routing via nginx (stableIngress pointant sur l'ingress existant).

Piege rencontre : le CRD Rollout exige le champ `protocol` explicite sur les ports de conteneur (ServerSideApply rejette l'omission, contrairement a un Deployment classique qui a une valeur par defaut).

Validation : premiere sync sans canary (un seul ReplicaSet, normal car pas de version precedente a comparer). Sur un changement de variable d'environnement, le canary s'est deroule correctement : 20% puis pause, 50% puis pause (capture ci-dessous), 100%, avec bascule complete et ScaleDown de l'ancien ReplicaSet.

Capture a l'etape 50% :
Step: 3/5, SetWeight: 50, ActualWeight: 50, Status: Paused (CanaryPauseStep)
revision 2 (canary) : 1 pod Running
revision 1 (stable) : 2 pods Running

## Etape 6 -- Pilotage canary manuel (pause, promote, abort)

Steps modifies pour permettre un pilotage manuel : setWeight 10, pause indefinie ({}), setWeight 50, pause 1m, setWeight 100.

### Scenario 1 -- Promotion normale

Commande : kubectl argo rollouts promote annuaire-dev-annuaire -n devhub-dev

Avant : Paused, Step 1/5, SetWeight 10, ActualWeight 10 (revision canary 1 pod, stable 2 pods)
Apres : le canary avance au step suivant (50%, pause 1 min automatique), puis termine a 100% Healthy.

Observation : promote ne fait avancer que d'un cran (la pause suivante s'applique normalement). Le canary garde son rythme prevu.

### Scenario 2 -- Annulation explicite (abort)

Commande : kubectl argo rollouts abort annuaire-dev-annuaire -n devhub-dev

Avant : Paused, Step 1/5, SetWeight 10 (revision canary 1 pod, stable 2 pods)
Apres : Status Degraded, message "RolloutAborted: Rollout aborted update to revision X", SetWeight et ActualWeight retombent a 0. Le ReplicaSet canary est scale a 0 puis supprime, le stable reprend 100% du trafic immediatement.

Un git revert du commit qui avait declenche ce canary a ete necessaire pour remettre Git en coherence avec l'etat reel du cluster (sinon ArgoCD aurait retente le meme canary au sync suivant).

### Scenario 3 -- Promotion forcee (promote --full)

Commande : kubectl argo rollouts promote annuaire-dev-annuaire -n devhub-dev --full

Avant : Paused, Step 1/5, SetWeight 10
Apres : SetWeight saute directement a 100, toutes les pauses et etapes intermediaires sont ignorees. Le Rollout passe par un etat transitoire Progressing (nouveaux pods en ContainerCreating) puis devient Healthy une fois la nouvelle ReplicaSet stable.

### Reponse argumentee : quand un promote --full est-il acceptable en production ?

Un promote --full saute toute observation intermediaire -- aucune chance de detecter un probleme avant que 100% du trafic soit bascule. C'est acceptable seulement dans deux cas : (1) en incident ou l'on sait deja avec certitude que le canary est bon (deja valide par un autre canal, comme un environnement de staging identique testé juste avant), et que le risque de rester sur l'ancienne version est pire que celui de sauter les paliers ; (2) en situation d'urgence ou chaque minute supplementaire sur une version buguee coute plus cher que le risque d'un déploiement non observe (ex: faille de securite critique a patcher immediatement).

Precautions a prendre : ne jamais l'utiliser par defaut ou par automatisme, le reserver a une decision humaine explicite et documentee (qui a decide, pourquoi), et toujours garder la possibilite d'un rollback immediat (le revert Git doit etre pret a etre push en quelques secondes). Idealement, journaliser cet usage pour audit -- un promote --full frequent est un signal que les analyses automatiques (etape 7) ne sont pas assez fiables ou que les paliers sont mal calibres.
