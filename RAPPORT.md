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

## Etape 7 -- AnalysisTemplate : promotion sur preuve

AnalysisTemplate avec 2 metriques Prometheus :
- error-rate : taux d'erreur 5xx du canary < 1% (requete avec fallback `or vector(0)` pour eviter les resultats vides)
- latency-p95 : latence p95 du canary < 300ms (meme pattern de fallback)

Parametres : interval 30s, count 10 (5 min d'observation), failureLimit 1.

### Cas nominal (FAIL_RATE=0)

Trafic genere en continu via curl en boucle. L'AnalysisRun a collecte 20 mesures (10 par metrique), toutes reussies. Le canary a ete automatiquement promu de 25% a 50% puis 100% sans intervention humaine.

Capture : AnalysisRun Successful, 20/20 mesures.

### Cas degrade (FAIL_RATE=0.5)

FAIL_RATE=0.5 dans values-dev.yaml : 50% des requetes retournent une 500. L'AnalysisRun a detecte le taux d'erreur excessif apres quelques mesures (6 succes, 2 echecs > failureLimit de 1) et a automatiquement annule le canary.

Message : "Metric error-rate assessed Failed due to failed (2) > failureLimit (1)"
Le Rollout est passe en Degraded, SetWeight a 0, le stable a repris 100% du trafic.

### Piege rencontre

Les requetes PromQL sans fallback (`or vector(0)`) provoquent une erreur "slice index out of range" quand le canary n'a pas encore recu de trafic. La fenetre de rate a ete elargie de 1m a 2m et des fallbacks ajoutes pour garantir un resultat meme sans donnees.

## Etape 8 -- Blue/Green sur planning

Migration du chart planning : rollout.yaml avec strategy.blueGreen, service-active.yaml, service-preview.yaml, ingress-preview.yaml (planning-preview.devhub.local).

Configuration : autoPromotionEnabled false, scaleDownDelaySeconds 300 (5 min pour permettre un rollback instantane).

Observation : au deploiement, 4 pods tournent simultanement (2 anciens active + 2 nouveaux preview). Le trafic utilisateur reste sur l'ancienne version via activeService, la nouvelle est accessible uniquement via previewService. Apres promotion, le activeService bascule sur la nouvelle version et l'ancien ReplicaSet reste actif pendant scaleDownDelaySeconds avant d'etre supprime.

### Comparatif canary vs blue/green

| Critere | Canary | Blue/Green |
|---|---|---|
| Exposition au risque | Progressive (10%, 25%, 50%...) | Tout ou rien (0% ou 100%) |
| Cout en ressources | Faible (1 pod canary) | Double (2x les replicas pendant la bascule) |
| Rollback | Scale down canary, stable intact | Repointer activeService sur ancien RS |
| Observation | Metriques en conditions reelles de trafic | Test interne sur previewService avant bascule |
| Cas d'usage ideal | APIs avec beaucoup de trafic, detection progressive | Migrations de schema, changements incompatibles, besoin de test complet avant exposition |

Piege rencontre : ArgoCD avec selfHeal:true re-synce le Rollout pendant la phase Paused, ce qui declenche la promotion automatiquement. En production, il faudrait configurer ArgoCD pour ignorer certains champs du Rollout (annotation argoproj.io/sync-options: Prune=false).

## Etape 9 -- Routage header-based (X-Beta-User)

Ajout de additionalIngressAnnotations dans la section trafficRouting.nginx du Rollout annuaire :
- canary-by-header: X-Beta-User
- canary-by-header-value: "true"

Argo Rollouts genere automatiquement un ingress canary avec ces annotations. Resultat :
- curl http://annuaire.devhub.local/healthz -> repond via le stable (200 OK)
- curl -H "X-Beta-User: true" http://annuaire.devhub.local/healthz -> repond via le canary (200 OK)

Le header a priorite sur le canary-weight : meme si le poids est a 25%, toute requete avec X-Beta-User: true est systematiquement envoyee au canary.

Usage metier : cela permettrait a l'equipe produit de tester chaque release sur leurs propres comptes avant n'importe quel utilisateur, en ajoutant simplement le header via une extension navigateur ou un proxy interne. Combine avec un AnalysisTemplate, on pourrait laisser l'equipe produit valider manuellement la preview pendant que les metriques automatiques surveillent le trafic reel en parallele.

## Etape 10 -- Alerting Alertmanager et notifications

### PrometheusRules

Deux regles creees dans le chart annuaire :

1. AnnuaireHighErrorRate (severity: page) : taux d'erreur 5xx > 1% pendant 5 min. Declencherait une alerte urgente (reveille l'astreinte).
2. AnnuaireHighLatency (severity: ticket) : latence p95 > 300ms pendant 30 min. Declencherait un ticket non urgent (traitement en heures ouvrees).

### Alertmanager routing

Configuration dans kube-prometheus-stack-values.yaml :
- route par defaut vers webhook-default
- match severity=page vers webhook-page
- match severity=ticket vers webhook-ticket
- group_by alertname + severity, group_wait 10s, repeat_interval 1h

Les trois receivers pointent vers le meme webhook.site (limitation de la version gratuite). En production, page irait vers PagerDuty/Opsgenie, ticket vers Jira/Slack.

### Validation

Les deux alertes sont visibles dans Prometheus /alerts, statut Inactive (normal car le service est sain). En cas de FAIL_RATE=0.5, l'alerte AnnuaireHighErrorRate passerait en Firing apres 5 min et Alertmanager posterait le payload JSON sur le webhook configure.

## Etape 11 -- Comparaison Argo Rollouts vs Flagger vs RollingUpdate

| Critere | Argo Rollouts | Flagger | RollingUpdate natif K8s |
|---|---|---|---|
| Installation | CRD + controleur dedie | CRD + controleur dedie | Rien (natif) |
| Canary | Oui (weight-based, nginx/istio) | Oui (mesh-based, istio/linkerd/nginx) | Non |
| Blue/Green | Oui (activeService/previewService) | Oui (primary/canary) | Non |
| Header-based routing | Oui (via nginx annotations) | Oui (via mesh) | Non |
| AnalysisTemplate | Oui (Prometheus, Datadog, etc.) | Oui (Prometheus, Datadog, etc.) | Non |
| Integration ArgoCD | Native (meme ecosysteme Argo) | Necesssite adaptation | Native (Deployment standard) |
| Rollback automatique | Oui (sur echec analyse) | Oui (sur echec analyse) | Uniquement sur crash pod |
| Complexite | Moyenne | Moyenne-haute (necessite mesh) | Faible |
| Observabilite du deploiement | Dashboard dedie + CLI | Grafana dashboard | kubectl rollout status |

### Mon choix pour ce TP

Argo Rollouts s'impose ici car on utilise deja ArgoCD (meme ecosysteme, meme CLI, meme philosophie GitOps). Flagger aurait necessite un service mesh (Istio ou Linkerd) qu'on n'a pas, ce qui aurait complexifie l'infrastructure pour un gain marginal dans notre contexte. Le RollingUpdate natif est insuffisant des qu'on veut du canary ou du blue/green avec observation.

En production avec un service mesh deja en place, Flagger serait un choix valide car il s'integre mieux avec les fonctionnalites avancees du mesh (circuit breaking, retries, mutual TLS). Sans mesh, Argo Rollouts avec nginx est le meilleur compromis simplicite/puissance.

## Etape 12 -- Synthese obligatoire

### Matrice comparatif detaillee

| Critere | RollingUpdate natif | Argo Rollouts | Flagger |
|---|---|---|---|
| Courbe d'apprentissage | 5 - rien a apprendre, natif K8s | 3 - CRD + CLI a maitriser, bien documente | 2 - necessite aussi un mesh (Istio/Linkerd) |
| Integration ArgoCD | 4 - Deployment standard, zero config | 5 - meme ecosysteme, CRD reconnu nativement | 2 - necessite des adaptations custom |
| Integration Flux | 4 - Deployment standard | 2 - pas d'integration native, necessite des hooks | 5 - concu pour fonctionner avec Flux |
| Variete des strategies | 1 - RollingUpdate uniquement | 5 - canary, blueGreen, experiment, analyse | 4 - canary, blueGreen, A/B via mesh |
| Variete des metric providers | 0 - aucun | 5 - Prometheus, Datadog, NewRelic, Wavefront, etc. | 4 - Prometheus, Datadog, CloudWatch |
| UI / dashboard | 1 - kubectl rollout status | 4 - dashboard web dedie + CLI riche | 2 - pas de dashboard dedie, Grafana dashboards |
| Cout operationnel cluster | 5 - zero, natif | 3 - un controleur + CRDs, leger | 2 - controleur + mesh obligatoire = lourd |
| Adapte a un mesh | 1 - pas de traffic shaping | 3 - supporte Istio/nginx/ALB | 5 - concu pour les mesh, traffic shaping natif |
| Communaute / releases | 5 - maintenu par K8s core | 4 - communaute Argo active, releases regulieres | 3 - communaute plus petite, maintenu par Fluxcd |
| Risque si controleur tombe | 5 - pas de controleur externe | 2 - le canary se fige a l'etape en cours, pas de promotion ni rollback possible | 2 - meme probleme, le canary se fige |

### Retrospective TP2 vers TP3

| Operation | TP2 (ArgoCD seul) | TP3 (ArgoCD + Rollouts + Prometheus) | Ressenti |
|---|---|---|---|
| Deployer une nouvelle version | git push, ArgoCD applique | git push, canary demarre automatiquement | Plus rassurant, on voit le trafic basculer progressivement |
| Rollback | git revert + push | Automatique si l'analyse echoue | Nettement mieux, pas besoin d'intervention humaine |
| Detecter un probleme | Regarder les pods manuellement | Prometheus detecte, alerte, et annule le canary | Le systeme reagit avant meme qu'on s'en rende compte |
| Observer ce qui se passe | UI ArgoCD (sync status) | Grafana dashboards + Prometheus metriques RED | Beaucoup plus riche, on voit le comportement reel du service |
| Ajouter un service | Un YAML dans platform/apps | Un YAML + rollout.yaml + services + analysistemplate | Plus complexe mais le gain en securite est enorme |
| Hotfix en urgence | git push direct | Le canary impose quand meme les paliers | Plus contraignant, promote --full existe mais c'est un choix explicite |

2 operations ou le surcout du TP3 n'est pas justifie dans une startup de 3 personnes :
1. La mise en place de l'AnalysisTemplate avec Prometheus -- pour 3 devs qui connaissent leur code, un simple canary avec pause manuelle suffit. Le temps de setup et de maintenance des requetes PromQL ne vaut pas le coup.
2. Le Blue/Green qui double les ressources -- dans une startup avec un budget cloud serre, doubler les pods a chaque deploiement est un luxe. Un RollingUpdate avec un bon healthcheck fait le travail.

L'operation qui justifie le passage du TP2 au TP3 pour un responsable plateforme en PME : le rollback automatique sur analyse. Quand on a 10 services et des deploiements quotidiens, un seul incident en production evite grace a l'AnalysisTemplate rembourse tout l'investissement de setup. C'est une assurance qui se paye une fois et qui protege en continu.

### Ce que cette chaine ne sait toujours pas faire

1. Tracabilite distribuee (OpenTelemetry, Jaeger, Tempo)
Risque : si un appel utilisateur traverse annuaire puis planning puis notif, on ne peut pas reconstituer la chaine. En cas de latence, impossible de savoir quel service est le goulot.
Outil : OpenTelemetry Collector + Grafana Tempo. Chaque service instrumente ses appels avec des trace-id propages via headers.
Ref : https://opentelemetry.io/docs/

2. Logs centralises (Loki, Fluent Bit)
Risque : les logs sont dans chaque pod individuellement. Si un pod crashe, ses logs disparaissent. Correler un pic d'erreur Prometheus avec les logs applicatifs correspondants est manuel et lent.
Outil : Fluent Bit en DaemonSet qui collecte les logs et les envoie vers Grafana Loki. Les exemplars Prometheus permettent de lier une metrique a un trace-id.
Ref : https://grafana.com/docs/loki/latest/

3. Mesure cote utilisateur (RUM, Web Vitals)
Risque : Prometheus mesure la latence cote serveur. Si le CDN est lent ou le navigateur rame, on ne le voit pas. L'experience reelle de l'utilisateur nous echappe.
Outil : Grafana Faro ou un SDK RUM qui remonte les Core Web Vitals (LCP, FID, CLS) vers un backend d'observabilite.
Ref : https://grafana.com/docs/faro/latest/

4. Chaos engineering (Chaos Mesh, LitmusChaos)
Risque : on ne teste jamais la resilience en conditions degradees. Un noeud qui tombe, un reseau lent, un disque plein -- on decouvre les problemes en production.
Outil : Chaos Mesh pour injecter des pannes controlees (kill pod, network delay, disk fill) et verifier que les alertes se declenchent et que les rollbacks fonctionnent.
Ref : https://chaos-mesh.org/docs/

5. Politique d'admission (Kyverno, OPA Gatekeeper)
Risque : rien n'empeche de deployer une image :latest, un container root, ou un pod sans limites de ressources. ArgoCD applique tout ce qui est dans Git sans validation.
Outil : Kyverno avec des ClusterPolicy qui bloquent les manifests non conformes avant meme qu'ils soient appliques.
Ref : https://kyverno.io/docs/

6. Signature des images (Sigstore, cosign)
Risque : n'importe qui avec un acces push au registry peut remplacer une image par une version compromise. On deploie sans verifier l'authenticite.
Outil : cosign pour signer les images en CI, et une politique Kyverno qui refuse les images non signees.
Ref : https://docs.sigstore.dev/

7. Backup et disaster recovery (Velero)
Risque : si le cluster explose ou si quelqu'un supprime un namespace par erreur, ArgoCD peut recreer les Deployments mais pas les donnees (PVC, bases de donnees).
Outil : Velero pour des snapshots reguliers des PVC et des resources K8s, avec restauration testee periodiquement.
Ref : https://velero.io/docs/

### Position d'architecte

Demain je deviens responsable plateforme pour 10 services et 30 developpeurs. Voici ma chaine :

Je garde ArgoCD comme socle GitOps -- c'est le pilier, tout passe par Git, zero kubectl en production. Je garde Argo Rollouts avec canary et AnalysisTemplate sur les services critiques (API publiques, services de paiement), mais je laisse un simple RollingUpdate sur les services internes a faible risque (backoffice, outils internes) pour ne pas imposer une complexite inutile partout.

Je remplace le kube-prometheus-stack installe a la main par une stack Grafana managee (Grafana Cloud ou equivalent) pour reduire la charge operationnelle de maintenance. Le monitoring ne doit pas etre un service de plus a maintenir.

J'ajoute en priorite : (1) Kyverno pour bloquer les images :latest et les containers root des le premier jour, (2) Sealed Secrets pour que les secrets soient enfin dans Git, et (3) OpenTelemetry + Loki pour avoir la tracabilite et les logs centralises avant le premier incident serieux. Velero vient juste apres, des qu'on a des donnees persistantes.

Le Blue/Green, je le reserve aux cas ou un canary progressif n'est pas possible (migration de schema de BDD, changement d'API incompatible). Pour tout le reste, le canary avec analyse automatique est le meilleur ratio securite/complexite.
