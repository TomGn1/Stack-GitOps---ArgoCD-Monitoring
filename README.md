Mise en place d'une stack de monitoring pour superviser mon homelab hébergé chez un cloud provider.

## I. Technologies utilisées :

**Orchestration** :
- Kubernetes : k3s

**GitOps** :
- Argo CD

**Gestion des secrets** :
- Sealed Secrets

**Observabilité** :
- Prometheus
- Alertmanager
- Grafana

## II. Compétences acquise :

**Pratique Kubernetes** :
- Déploiement d'un cluster
- Application des concepts liés au services k8s
- Découverte des _Helm_ Charts
**Approfondissement sur les principes du CI/CD** :
- Découverte d'_Argo CD_ pour déployer des applicatifs sur cluster
- Découverte de la gestion des secrets avec _Argo CD_
**Approfondissement sur la mise en place d'outils de supervision** :
- Mise en place d'une stack _Prometheus_ + _Alertmanager_ + _Grafana_

## III. Architecture

### 1. Choix architecturaux

**Séparation par responsabilité** : Cluster k3s dédié, pour la plateforme contenant les outils de pilotage/observabilité.
	- La plateforme peut être maintenue/redémarrée sans impacter les autres clusters.

**Dimensionnement du cluster** : 
Une seule node :
- 2 vCPU / 4Go RAM
- Pas de HA

### 2. Choix techniques et compromis

- **Argo CD** : 
	- Outils de delivery, une fois les pods déployés il n'est plus dans la boucle.
	- Peut se géré lui-même (pattern **App of Apps**)
	- Une seule commande pour tout déployer/restaurer
_Argo CD_ détecte les fichiers dans _Git_ et déploie tout seul (y compris ses propres mises à jour). Idéal pour la reproductibilité et la résilience.

- **Stack d'observabilité** :
	- _Prometheus_ : Scrape les métriques de pods les _ServiceMonitors_ (CRD); évalue les règles d'alertes via _PrometheusRules_(CRD)
	- _Grafana_ : Visualisation des métriques et dashboards avec les informations renvoyées par _Prometheus_.
	- _Alertmanager_ : Permet de gérer les alertes; il reçoit les alertes de _Prometheus_ pour les router vers un service tier (Slack, mail, etc.)

- **Gestion des secrets** :
	- Utilisation se _Sealed Secrets_, car ce homelab n'a pas de Vault existant. Pour ce projet il n'est pas nécessaire d'introduire de dépendances non justifiées.

### 3. Structure logique k8s

- 1 applicatif par pod : _Helm_ déploie chaque programme dans son propre pod.
- 1 seule chart _Helm_ qui embarque la stack d'observabilité.

_Exemple de structure_ :
```bash
Deployment "argocd-server"    →  Pod  →  Conteneur : argocd-server
Deployment "grafana"          →  Pod  →  Conteneur : grafana
StatefulSet "prometheus"      →  Pod  →  Conteneur : prometheus
StatefulSet "alertmanager"    →  Pod  →  Conteneur : alertmanager
```

- Les services assurent la communication entre les pods et vers l'extérieur :

**Communication** :
```bash
Pod vers pod : ClusterIP
Admin vers UIs : NodePort/Ingress
```

Communication interne _Grafana_ vers _Prometheus_ :
DNS : `http://prometheus-operated.monitoring.svc.cluster.local:9090`
