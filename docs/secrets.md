## I. Inventaire des secrets

| Nom                           | Namespace      | Contenu                      | Utilisé par                            |
| ----------------------------- | -------------- | ---------------------------- | -------------------------------------- |
| `grafana-admin-secret`        | `monitoring`   | `admin-user, admin-password` | Grafana                                |
| `alertmanager-smtp-secret`    | `monitoring`   | `smtp_auth_password`         | Alertmanager                           |
| `letsencrypt-prod`            | `cert-manager` | clé privée ACME              | cert-manager                           |
| `argocd-initial-admin-secret` | `argocd`       | `password`                   | ArgoCD (supprimé après initialisation) |

## II. Recréer un secret

Si un _SealedSecret_ doit être recréé (compromission, expiration, etc.) :

1. Créer le secret en clair (temporaire, ne jamais commité) :

```bash
kubectl create secret generic grafana-admin-secret \
  --namespace monitoring \
  --from-literal=admin-password='motDePasse' \
  --from-literal=admin-user='admin' \
  --dry-run=client -o yaml > /tmp/secret.yaml
```

2. Chiffrer avec `kubeseal` :

```bash
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  < /tmp/secret.yaml \
  > platform/monitoring/grafana-sealed-secret.yaml
```

3. Supprimer le fichier en clair :

```bash
rm /tmp/secret.yaml
```

4. Commiter et pusher :

```bash
git add platform/monitoring/<nomDuSecret>.yaml 
git commit -m "chore: rotate <nomDuSecret>" 
git push
```

>[!NOTE] 
>Après rotation, forcer le redémarrage du `StatefulSet` si le secret est utilisé par _Alertmanager_ ou _Prometheus_, voir [`troubleshooting.md`](./troubleshooting.md).
## Clé de chiffrement Sealed Secrets

La clé privée du controller _Sealed Secrets_ est générée automatiquement à l'installation. Elle est stockée dans `kube-system` et n'est **jamais commitée dans Git**.

>[!WARNING]
>Si le cluster est recréé sans sauvegarder cette clé, tous les _SealedSecrets_ existants dans le repo deviennent **illisibles et irrécupérables**. Il faudra recréer tous les secrets depuis zéro.

- Sauvegarder la clé :

```bash
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key.yaml
```

>[!WARNING]
>Ce fichier contient la clé privée en clair. Ne jamais le commiter dans _Git_. Le stocker dans un endroit sécurisé (coffre-fort de mots de passe, stockage chiffré).

- Restaurer la clé sur un nouveau cluster avant d'installer _Sealed Secrets_ :

```bash
kubectl apply -f sealed-secrets-master-key.yaml
```