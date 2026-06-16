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
