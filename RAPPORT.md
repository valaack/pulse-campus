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
