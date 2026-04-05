## I. Prérequis

- K3s opérationnel :

```bash
k3s --version
```

- `kubectl` configuré :

```bash
kubectl cluster-info
```

- _Helm_ installé :

```bash
helm version
```

- Configuration du NAT (dans ce projet, le cluster se trouve derrière un double NAT : _Proxmox_ _iptables_ + _pfSense_ Port Forward) :

>[!NOTE]
>**Particularité derrière un Double NAT** :
>
>Lors de l'émission d'un certificat Let's Encrypt, _cert-manager_ effectue un **self-check** : il tente de joindre lui-même le domaine (ex: `grafana.morguen.fr`) via l'IP publique pour vérifier que le challenge HTTP-01 est accessible avant de contacter _Let's Encrypt_.
>
>Dans un setup derrière un double NAT (ici, _Proxmox_ + _pfSense_), ce trafic part du cluster vers l'IP publique puis tente de revenir vers le cluster, c'est ce qu'on appelle le **hairpin NAT**. Ce type de trafic est généralement bloqué ou mal routé.
>
>En ajoutant les entrées DNS dans `NodeHosts`, _CoreDNS_ résout directement `grafana.morguen.fr` vers l'IP interne de _Traefik_ (`10.0.0.15`) sans passer par l'IP publique. Le self-check de cert-manager reste dans le réseau interne et le challenge HTTP-01 s'effectue correctement.
>
>- Editer le _CoreDNS_ :
>
>```bash
>kubectl edit configmap coredns -n kube-system
>```
>
>- Ajouter les entrées désirées dans la partie `NodeHosts` :
>
>```yaml
>NodeHosts: |
>    <ipK3s> grafana.<nomDeDomaine>.fr
>    <ipK3s> argocd.<nomDeDomaine>.fr
>```
>
>- Pour que cette configuration ne soit pas écrasée au redémarrage de K3s, il faut la rendre persistante :
>```bash
>sudo nano /var/lib/rancher/k3s/server/manifests/coredns-custom.yaml
>```
>```yaml
>apiVersion: v1
>kind: ConfigMap
>metadata:
>  name: coredns
>  namespace: kube-system
>data:
>  NodeHosts: |
>    <ipK3s> <nomK3s>
>    <ipK3s> grafana.<nomDeDomaine>.fr
>    <ipK3s> argocd.<nomDeDomaine>.fr
>```

- Configuration des entrées DNS (A/AAAA vers IP publique).

- Compte _GitHub_, tunnel SSH configuré entre le serveur K3s et le repo.

## II. Bootstrap

### 1. cert-manager

- Ajouter le repo _Helm_ _cert-manager_ :

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

- Installation de _cert-manager_ avec ses CRDs :

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

- Vérifier que tout est monté :

```bash
kubectl get pods -n cert-manager
```

---
### 2. ClusterIssuer

- Création du `ClusterIssuer`, `platform/argocd/cluster-issuer.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

---
### 3. ArgoCD

#### 3.1 Installation :

- Installer _Argo CD_ :

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f platform/argocd/values.yaml
```

- Récupérer le mot de passe admin initial :

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

- Connexion avec les identifiants suivants :
	- **Login** : admin
	- **Password** : le résultat de la commande ci-dessus

- Une fois connecté, changer le mot de passe :

```text
User Info (en haut à gauche) → Update Password
```

- Supprimer le secret initial qui n'a plus besoin d'exister :

```bash
kubectl delete secret argocd-initial-admin-secret -n argocd
```

#### 3.2 Connexion au repo Git :

Il est possible de connecter _Argo CD_ au repo de deux façons :

**Via manifest** :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitops-homelab-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: git@github.com: <nomUtilisateurGitHub>/<nomDuRepoGit>
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
```

**Via l'UI** :
```text
Settings → Repositories → Connect Repo
→ Via HTTPS ou SSH
→ Coller l'URL de ton repo GitHub
```

- **Dans l'UI ArgoCD :**

```text
Settings
  → Repositories
    → Connect Repo
```

- Remplir les champs suivant :

```text
Connection Method : HTTPS
Project           : default
Repository URL    : https://github.com/<nomUtilisateurGitHub>/<nomDuRepoGit>
```

>[!NOTE]
Laisser les champs username/password vides le est repo public, aucune authentification est nécessaire.

- Initialiser la connexion le statut doit passer **Successful** en vert ✅.
#### 3.3 Création de l'app

- Création de la première app, `apps/monitoring-app.yaml` :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: "56.6.2"
      helm:
        valueFiles:
          - $values/platform/monitoring/values.yaml
    - repoURL: https://github.com/<nomUtilisateurGitHub>/<nomDuRepoGit>
      targetRevision: HEAD
      ref: values
    - repoURL: https://github.com/<nomUtilisateurGitHub>/<nomDuRepoGit>
      targetRevision: HEAD
      path: platform/monitoring
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

>[!NOTE]
>Ceci est une version finale du manifest. Les sections _Sealed Secrets_ et _kube-prometheus-stack_ doivent être bootstrappées avant de l'appliquer.

- Créer le `values.yaml` pour kube-prometheus-stack :

```yaml
grafana:
  enabled: true
  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - grafana.<nomDeDomaine>.fr
    tls:
      - secretName: grafana-tls
        hosts:
          - grafana.<nomDeDomaine>.fr
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod

prometheus:
  prometheusSpec:
    retention: 15d

alertmanager:
  enabled: true
```

- Push vers le repo :

```bash
cd ~/gitops-homelab
git add .
git commit -m "feat: add monitoring Application and values"
git push origin main
```

- **Ajouter le repo Helm dans le cluster** car _Argo CD_ en a besoin pour résoudre la chart. Créer un secret avec le repo _Helm_ :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  helm.repositories: |
    - url: https://prometheus-community.github.io/helm-charts
      name: prometheus-community
```

- Appliquer l'application dans le cluster :

```bash
kubectl apply -f apps/monitoring-app.yaml
```

- Vérifier le statut dans l'UI _Argo CD_  :

```text
Applications
  └── monitoring  → statut "Syncing..." puis "Healthy" ✅
```

---
### 4. Sealed Secrets

- Installation de _Sealed Secrets_ :

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system
```

- Vérifier que le pod tourne :

```bash
kubectl get pods -n kube-system | grep sealed-secrets
```

- Installer _kubeseal_ sur le serveur K3s :

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.24.0-linux-amd64.tar.gz
sudo mv kubeseal /usr/local/bin/
```

- Vérification :

```bash
kubeseal --version
```

- Création du secret en clair (temporaire, jamais commité) :

```bash
kubectl create secret generic grafana-admin-secret \
  --namespace monitoring \
  --from-literal=admin-password='motDePasse' \
  --from-literal=admin-user='admin' \
  --dry-run=client \
  -o yaml > /tmp/grafana-secret.yaml
```

- Chiffrer avec _kubeseal_ :

```bash
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  < /tmp/grafana-secret.yaml \
  > platform/monitoring/grafana-sealed-secret.yaml
```

>[!WARNING]
>**Problème rencontré** : le controller _Sealed Secrets_ ne s'appelle pas `sealed-secrets-controller` par défaut avec cette installation, il faut préciser `--controller-name=sealed-secrets` et `--controller-namespace=kube-system`.
>
>Pour vérifier le nom du controller :
>```bash
>kubectl get pods -n kube-system | grep sealed
>```

- Supprimer le fichier en clair :

```bash
rm /tmp/grafana-secret.yaml
```

- Vérifier le fichier chiffré :

```bash
cat platform/monitoring/grafana-sealed-secret.yaml
```

- Mise à jour du manifest `values.yaml` _Grafana_ pour qu'il pointe vers ce secret au lieu du mot de passe en clair :

```yaml
grafana:
  enabled: true
  admin:
    existingSecret: grafana-admin-secret
    userKey: admin-user
    passwordKey: admin-password
  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - grafana.<nomDuDomaine>.fr
    tls:
      - secretName: grafana-tls
        hosts:
          - grafana.<nomDuDomaine>.fr
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
```

>[!NOTE]
Suppression de `adminPassword` pour dire à _Grafana_ d'aller lire le secret Kubernetes.

_Argo CD_ va détecter le changement et déployer le _Sealed Secret_. _Sealed Secrets controller_ va le déchiffrer automatiquement et créer le Secret Kubernetes. 

- Ajouter au manifest `apps/monitoring-app.yaml` le chemin `platform/monitoring` pour qu'il déploie les manifests de ce dossier :

```yaml
spec:
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: "56.6.2"
      helm:
        valueFiles:
          - $values/platform/monitoring/values.yaml
    - repoURL: https://github.com/<nomUtilisateurGitHub>/<nomDuRepoGit>
      targetRevision: HEAD
      ref: values
    - repoURL: https://github.com/<nomUtilisateurGitHub>/<nomDuRepoGit>
      targetRevision: HEAD
      path: platform/monitoring
```

_Argo CD_ gérera automatiquement le _Sealed Secret_, plus besoin d'utiliser manuellement la commande `kubectl apply`.

- Commiter :

```bash
git add platform/monitoring/grafana-sealed-secret.yaml
git add platform/monitoring/values.yaml
git add apps/monitoring-app.yaml
git commit -m "feat: use sealed secret for grafana credentials and add monitoring manifests path to ArgoCD application"
git push
```

---
### 5. Configuration Alertmanager

#### 5.1 Mot de passe d'application Gmail

```text
myaccount.google.com
  → Sécurité et connexion
    → Validation en deux étapes (doit être activée)
      → Mots de passe des applications
        → Sélectionner une application : "Autre (nom personnalisé)"
        → Nom : "Alertmanager homelab"
        → Générer
```

Ce mot de passe est une donnée sensible, il faut créer un _SealedSecret_ pour les credentials SMTP :

```bash
kubectl create secret generic alertmanager-smtp-secret \
  --namespace monitoring \
  --from-literal=smtp_auth_password='<motDePasseAppGmail>' \
  --dry-run=client -o yaml > /tmp/smtp-secret.yaml

kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  < /tmp/smtp-secret.yaml \
  > platform/monitoring/alertmanager-sealed-secret.yaml

rm /tmp/smtp-secret.yaml
```

- Commiter le _SealedSecret_ pour qu'_Argo CD_ puisse le déchiffrer

```bash
git add platform/monitoring/alertmanager-sealed-secret.yaml
git commit -m "add: Alertmanger SealedSecret"
git push
```

- Vérifier qu'_Alertmanager_ a redémarré avec la nouvelle configuration :

```bash
kubectl get pods -n monitoring | grep alertmanager
```

- Vérifier les logs _Alertmanager_ pour voir s'il y a des erreurs SMTP :

```bash
kubectl logs -n monitoring \
  alertmanager-monitoring-kube-prometheus-alertmanager-0 \
  -c alertmanager
```
#### 5.2 Configuration receivers / routes

- Description de la route : elle définit comment _Alertmanger_ distribue les alertes. `receiver: 'email'` indique que, par défaut, toues les alertes sont envoyées par mail. `group-by` permet de regrouper les alertes du même ensemble.

>[!NOTE]
>**La sous-route `Watchdog`**:
> 
>_Prometheus_ envoie en permanence une alerte fictive appelée "Watchdog" pour signaler qu'il est vivant. Elle est redirigée vers `null` pour la rendre silencieuse, car elle n'a aucune valeur opérationnelle.

```yaml
    route:
      receiver: 'email'          # ← receiver par défaut
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      routes:
        - receiver: 'null'       # ← Watchdog toujours silencieux
          matchers:
            - alertname = "Watchdog"
```

- Les receivers définissent les destinations possibles. Ici `null` est un receiver vide qui absorbe les alertes du `Watchdog`. `email` permet d'envoyer un second mail avec une notification de résolution.

```yaml
    receivers:
      - name: 'null'
      - name: 'email'
        email_configs:
          - to: 'email@gmail.com'
            send_resolved: true
```

- Ces règles ci-dessous, sont décrites dans le but d'éviter le bruit en supprimant les alertes `warning/info` quand il y a déjà une alerte `critical` sur le même namespace.

```yaml
    inhibit_rules:
      - target_matchers:
          - severity =~ warning|info
        source_matchers:
          - severity = critical
        equal: ['namespace', 'alertname']
```

#### 5.3 Exclusion des composants intégrés à K3s

Dans une installation K3s standard, le `kubeControllerManager`, le `kubeScheduler` 
et le `kubeProxy` sont intégrés directement dans le binaire K3s. Ils ne sont pas 
exposés comme endpoints séparés. _Prometheus_ tente de les scraper et ne les trouvant 
pas, génère des alertes perpétuelles de type `KubeControllerManagerDown`.

Il faut donc désactiver leurs `ServiceMonitors` depuis le `values.yaml` :

```yaml
kubeScheduler:
  enabled: false

kubeControllerManager:
  enabled: false

kubeProxy:
  enabled: false
```

#### 5.4 Ajout d'Alertemanager au values.yaml 

```yaml
alertmanager:
  enabled: true
  config:
    global:
      resolve_timeout: 5m
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'email@gmail.com'
      smtp_require_tls: true
      smtp_auth_username: 'email@gmail.com'
      smtp_auth_password_file: /etc/alertmanager/secrets/alertmanager-smtp-secret/smtp_auth_password
    route:
      receiver: 'email'
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      routes:
        - receiver: 'null'
          matchers:
            - alertname = "Watchdog"
    receivers:
      - name: 'null'
      - name: 'email'
        email_configs:
          - to: 'email@gmail.com'
            send_resolved: true
    inhibit_rules:
      - target_matchers:
          - severity =~ warning|info
        source_matchers:
          - severity = critical
        equal: ['namespace', 'alertname']
  alertmanagerSpec:
    secrets:
      - alertmanager-smtp-secret
kubeScheduler:
  enabled: false

kubeControllerManager:
  enabled: false

kubeProxy:
  enabled: false
```

- Commit et Push :

```bash
git add platform/monitoring/values.yaml
git commit -m "fix: complete alertmanager config with email receiver"
git push
```

---
### 6. Déploiement de `monitoring-app`

#### 6.1 Déploiement 

Le déploiement est effectué avec cette unique ligne de commande :

```bash
kubectl apply -f apps/monitoring-app.yaml
```

#### 6.2 Vérification dans l'UI 

Dans l'UI _Argo CD_ le statut doit être : Syncing → Healthy

![screenshotArgoCdUi](./img/argoCdSyncReady.png)

#### 6.3 Vérification des pods

```bash
kubectl get pods -n monitoring
```

![screenshotMonitoringPods](./img/getMonitoringPods.png)
#### 6.4 Vérification des ingress et accès HTTPS Grafana

**Côté Cluster** :

- Vérifier que l'ingress est créé :

```bash
kubectl get ingress -n monitoring
```

![screenshotIngressMonitoring](./img/getIngressMonitoring.png)

- Vérifier que le certificat TLS est émis :

```bash
kubectl get certificate -n monitoring
```

![screenshotGrafanaTls](./img/grafanaTlsTrue.png)

- Vérifier que le secret TLS est présent :

```bash
kubectl get secret grafana-tls -n monitoring
```

![screenshotGrafanaTlsSecret](./img/grafanaTlsSecret.png)

**Côté accès** :

- Vérifier que le nom de domaine résout vers l'IP publique :

```bash
dig grafana.<nomDeDomaine>.fr
```

- Vérifier le certificat HTTPS :

```bash
curl -v https://grafana.<nomDeDomaine>.fr 2>&1 | grep -E "SSL|certificate|issuer"
```

#### 6.5 Vérifications sur Alertmanager

- Vérifier que le _SealedSecret_ SMTP est bien déchiffré :

```bash
kubectl get secret alertmanager-smtp-secret -n monitoring
```

- Envoyer une alerte de test manuellement

```bash
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager \
  -n monitoring 9093:9093 &

curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TestAlert","severity":"warning"}}]'
```

