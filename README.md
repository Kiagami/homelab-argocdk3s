# argocd-homelab

```
bootstrap/root-app.yaml        # einmalig per kubectl apply
apps/adguard.yaml              # ArgoCD Application
manifests/adguard/             # Kustomize: StatefulSet, Services, Ingress, PDB
manifests/adguard/sync/        # optional: adguardhome-sync
```

## Installation
1. Repo auf GitHub/Gitea pushen, `repoURL` in `bootstrap/root-app.yaml` und `apps/adguard.yaml` anpassen.
2. In `service-dns.yaml` eine freie MetalLB-IP, in `ingress.yaml` den Hostnamen setzen.
3. `kubectl apply -f bootstrap/root-app.yaml`
4. Pods starten, sind aber erst "Ready", wenn der Wizard durch ist.

## Erst-Einrichtung (pro Pod einmal)
```
kubectl -n adguard port-forward pod/adguard-0 3000:3000   # http://localhost:3000
kubectl -n adguard port-forward pod/adguard-1 3000:3000
kubectl -n adguard port-forward pod/adguard-2 3000:3000
```
Im Wizard: Admin-Interface Port **80**, DNS Port **53**, auf allen Interfaces.
Gleichen Benutzernamen/Passwort bei allen drei verwenden.

## Sync aktivieren
1. Secret `adguard-sync-credentials` anlegen (siehe `sync/secret.example.yaml`).
2. In `kustomization.yaml` die Zeile `sync/deployment.yaml` einkommentieren, committen.
Aenderungen nur noch in der UI von adguard-0 machen (Ingress zeigt dorthin).
# homelab-argocdk3s
