# homelab-argocdk3s

AdGuard Home läuft als StatefulSet mit **1 Replica** im Namespace `adguard`, die Daten liegen auf **Longhorn**-Volumes.
Die Admin-UI ist nur aus dem Heimnetz erreichbar (Traefik-Allowlist, TLS per cert-manager).
DNS läuft über MetalLB, nicht über Traefik.

```
bootstrap/root-app.yaml        # einmalig per kubectl apply
apps/adguard.yaml              # ArgoCD Application
manifests/adguard/             # Kustomize: StatefulSet, Services, Ingress
```

## Architektur

| Komponente | Datei | Zweck |
|---|---|---|
| StatefulSet `adguard` | `statefulset.yaml` | 1 Pod, PVCs `work` (1Gi) und `conf` (100Mi) auf Longhorn |
| Service `adguard-headless` | `service-ui.yaml` | Headless Service, wird von `serviceName` im StatefulSet benötigt |
| Service `adguard-ui` | `service-ui.yaml` | Web-UI (Port 80), wählt Pod `adguard-0` |
| Service `adguard-dns` | `service-dns.yaml` | LoadBalancer via MetalLB, `192.168.7.55`, Port 53 UDP/TCP |
| Ingress `adguard` | `ingress.yaml` | `adguard.kmsystems.ddns.net`, TLS, Allowlist-Middleware |
| Kustomization | `kustomization.yaml` | Setzt `namespace: adguard` für alle Ressourcen |

Wichtige IPs:

- `192.168.7.55`: DNS-Server für alle Clients (MetalLB, `adguard-dns`)
- `192.168.7.50`: Traefik-LoadBalancer (Ziel für interne Auflösung der Web-Namen)

Entscheidungen:

- **Ein Pod statt drei:** Es gibt genau ein Query-Log und eine Statistik, keine Replikation der Config nötig.
  Ein PodDisruptionBudget und `adguardhome-sync` gibt es deshalb nicht mehr.
- **Longhorn statt `local-path`:** Das Volume ist über mehrere Nodes repliziert, der Pod kann nach einem
  Node-Ausfall auf einem anderen Node starten.
- **`externalTrafficPolicy: Local`** am `adguard-dns` Service erhält die Client-IPs für die AdGuard-Statistik.
  MetalLB kündigt die IP von dem Node an, auf dem der Pod läuft.

## Deployment

Deployt wird ausschließlich über ArgoCD (Application `adguard`, Pfad mit der `kustomization.yaml`).
Nicht manuell mit `kubectl apply` arbeiten, sonst driftet der Zustand, und `selfHeal` überschreibt Änderungen.

Beispiel-Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: adguard
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<USER>/<REPO>.git
    targetRevision: main
    path: manifests/adguard
  destination:
    server: https://kubernetes.default.svc
    namespace: adguard
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  ignoreDifferences:
    - group: apps
      kind: StatefulSet
      jsonPointers:
        - /spec/volumeClaimTemplates
```

`ignoreDifferences` auf `volumeClaimTemplates` verhindert Dauer-OutOfSync bei StatefulSets.
`prune: true` sorgt dafür, dass aus Git entfernte Ressourcen auch im Cluster gelöscht werden.

## Installation (Neuaufbau)

1. Repo auf GitHub/Gitea pushen, `repoURL` in `bootstrap/root-app.yaml` und `apps/adguard.yaml` anpassen.
2. In `service-dns.yaml` eine freie MetalLB-IP, in `ingress.yaml` den Hostnamen setzen.
3. `kubectl apply -f bootstrap/root-app.yaml`
4. Der Pod startet, ist aber erst "Ready", wenn der Wizard durch ist (oder eine Config wiederhergestellt wurde).

## Ersteinrichtung (Setup-Wizard)

Vor dem Wizard lauscht AdGuard nur auf Port **3000**. Die `readinessProbe` prüft TCP 53,
deshalb ist der Pod bis zum Abschluss des Wizards `0/1 Ready`. Das ist erwartet.
Der Service hat in dieser Zeit keine Endpoints, Ingress und DNS funktionieren noch nicht.

```bash
kubectl -n adguard get pods
kubectl -n adguard port-forward pod/adguard-0 3000:3000
```

Läuft `kubectl` nur auf einem Node (kein lokales kubectl), zusätzlich vom Notebook:

```bash
ssh -L 3000:localhost:3000 root@<node>
```

Browser: `http://localhost:3000`

Im Wizard:

- Admin-Interface: alle Schnittstellen, Port **80**
- DNS-Server: alle Schnittstellen, Port **53**
- Benutzer und starkes Passwort setzen
- Die Seite "Configure your devices" ist nur eine Info-Seite, einfach **Next** und **Open Dashboard**

Nach dem Wizard schließt sich Port 3000 im Pod. `connection refused` auf 3000 heißt also: Pod ist bereits eingerichtet.

Prüfen:

```bash
kubectl -n adguard get pods -o wide
kubectl -n adguard get endpoints
kubectl -n adguard get svc adguard-dns   # EXTERNAL-IP = 192.168.7.55
kubectl -n adguard get pvc               # STORAGECLASS = longhorn, STATUS = Bound
```

Zugriff auf die UI danach über den Ingress oder per Port-Forward auf Port 80:

```bash
kubectl -n adguard port-forward pod/adguard-0 8080:80
```

Hinweis: `kubectl port-forward` braucht keine Pod- oder Node-IP. Es läuft über den API-Server
aus der kubeconfig, der Pod ist unabhängig vom Node erreichbar.

## Ingress, TLS und LAN-Beschränkung

Der Ingress liegt im Namespace `adguard`, nicht in `default`. Ein Ingress kann nur Services
im eigenen Namespace als Backend nutzen, und `adguard-ui` liegt in `adguard`.
Certificate und TLS-Secret (`adguard-tls-cert`) entstehen ebenfalls dort.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: adguard
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.middlewares: "kube-system-heimnetz-whitelist@kubernetescrd"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - adguard.kmsystems.ddns.net
      secretName: adguard-tls-cert
  rules:
    - host: adguard.kmsystems.ddns.net
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: adguard-ui
                port:
                  number: 80
```

- Die Middleware `heimnetz-whitelist` (Traefik v3, `ipAllowList`) liegt zentral in `kube-system`
  und wird per Annotation `<namespace>-<name>@kubernetescrd` referenziert.
- Das Zertifikat kommt per HTTP-01. Der temporäre Ingress `cm-acme-http-solver-*` von cert-manager
  ist normal und verschwindet nach der Ausstellung. Die Middleware wirkt auf ihn nicht.
- Die Allowlist greift nur, wenn Traefik die echte Client-IP sieht
  (`externalTrafficPolicy: Local` am Traefik-Service):
  ```bash
  kubectl -n kube-system get svc traefik -o jsonpath='{.spec.externalTrafficPolicy}'
  ```

## Interne Namensauflösung (wichtig)

`adguard.kmsystems.ddns.net` löst öffentlich auf die WAN-IP auf. Ohne Gegenmaßnahme geht ein
LAN-Client über den Router (Hairpin-NAT), Traefik sieht dann die WAN-IP als Absender und
antwortet mit **403**, weil sie nicht in der Allowlist steht.

Lösung: Im LAN muss der Name direkt auf Traefik zeigen.

- AdGuard: *Filters → DNS rewrites*, `adguard.kmsystems.ddns.net` auf `192.168.7.50`
  (optional Wildcard `*.kmsystems.ddns.net` auf `192.168.7.50` für alle Dienste)
- UniFi/DHCP verteilt `192.168.7.55` als DNS-Server (WAN-DNS und DHCP-DNS pro VLAN), ohne externen
  Fallback-DNS, sonst umgehen Geräte AdGuard und den Rewrite
- Kein dauerhafter `/etc/hosts`-Eintrag auf Clients (verdeckt später Fehler)

## Longhorn: Backup und Failover

**Backup**

- In der Longhorn-UI einen Recurring Job (Snapshot und/oder Backup auf ein Backup-Target) für die Volumes
  `conf-adguard-0` und `work-adguard-0` anlegen. Wichtig ist vor allem `conf`, dort liegt `AdGuardHome.yaml`
  (Benutzer, Filter, Rewrites, Clients).
- Manuelle Sicherung der Config:
  ```bash
  kubectl -n adguard cp adguard-0:/opt/adguardhome/conf/AdGuardHome.yaml ./AdGuardHome.yaml
  # Alternative:
  kubectl -n adguard exec adguard-0 -- cat /opt/adguardhome/conf/AdGuardHome.yaml > AdGuardHome.yaml
  ```
  Die Datei enthält den Passwort-Hash: **nicht ins Git**, sicher ablegen.

**Restore der Config in ein leeres `conf`-Volume**

```bash
kubectl -n adguard cp ./AdGuardHome.yaml adguard-0:/opt/adguardhome/conf/AdGuardHome.yaml
kubectl -n adguard delete pod adguard-0
```

Der Pod startet neu, liest die Config statt den Wizard zu starten und wird Ready. Query-Log und Statistik
liegen im `work`-Volume und sind nur über Longhorn-Backup/Snapshot wiederherstellbar.

**Failover bei Node-Ausfall**

- In den Longhorn-Einstellungen **Pod Deletion Policy When Node is Down** auf `delete-statefulset-pod`
  setzen. Sonst bleibt der StatefulSet-Pod auf dem toten Node in `Terminating` hängen und startet nicht neu.
- Auch dann dauert ein Failover einige Minuten (Erkennung, Volume detachen, Pod starten). In dieser Zeit
  ist DNS nicht erreichbar. Geplante Node-Drains sind meist schneller (unter einer Minute).
- In der Longhorn-UI prüfen, dass die Volumes mehrere gesunde Replicas haben (Standard: 3).

## Migration von 3 Replicas (`local-path`) auf 1 Replica (Longhorn)

Einmalige Aktion, schon durchgeführt. Zur Dokumentation, falls sie erneut nötig wird:
`volumeClaimTemplates` eines StatefulSets sind unveränderlich, die Migration geht deshalb nicht allein über Git.

1. Config von `adguard-0` sichern (siehe Backup oben).
2. Manifeste in Git anpassen (`replicas: 1`, `storageClassName: longhorn`, PDB und `sync/` entfernt,
   `kustomization.yaml` bereinigt), Änderung pushen.
3. Auto-Sync der ArgoCD-App pausieren.
4. Altes StatefulSet und die alten PVCs von Hand löschen:
   ```bash
   kubectl -n adguard delete statefulset adguard
   kubectl -n adguard delete pvc --all
   kubectl get pv | grep adguard   # Reste im Status Released löschen
   ```
5. In ArgoCD mit **Prune** synchronisieren. Das StatefulSet entsteht neu, PVCs kommen von Longhorn.
6. Config zurückkopieren und Pod neu starten (siehe Restore oben), Auto-Sync wieder aktivieren.
7. Aufräumen: `kubectl -n adguard get pdb,deploy` darf weder das PDB noch `adguardhome-sync` zeigen,
   sonst von Hand löschen. Das Secret `adguard-sync-credentials` kann entfernt werden.

Wichtig: Das alte PDB (`minAvailable: 2`) würde bei einem Pod jeden Node-Drain blockieren. Es muss weg sein.

## Troubleshooting

| Symptom | Ursache | Lösung |
|---|---|---|
| Pod `0/1 Ready`, Service ohne Endpoints | Wizard nicht durchlaufen, Probe prüft Port 53 | Wizard per Port-Forward auf 3000 abschließen oder Config wiederherstellen |
| Port-Forward auf Nodes landet im Nichts | Port 3000 ist in keinem Service veröffentlicht | `kubectl port-forward pod/adguard-0 3000:3000` nutzen |
| `connection refused` beim Port-Forward auf 3000 | Pod ist bereits eingerichtet (3000 geschlossen) | UI auf Port 80 nutzen |
| `localhost:3000` im Browser geht nicht | kubectl läuft auf dem Node, nicht auf dem Notebook | SSH-Tunnel `ssh -L 3000:localhost:3000 root@<node>` |
| `adguard-dns` EXTERNAL-IP `<pending>` | Kein LoadBalancer-Provider oder IP aus dem Pool nicht verfügbar | MetalLB-Pool und Annotation `metallb.universe.tf/loadBalancerIPs` prüfen |
| PVC bleibt `Pending` | Longhorn nicht bereit oder StorageClass falsch | `kubectl get sc`, Longhorn-UI und `describe pvc` prüfen |
| PVC hängt in `Terminating` | Pod nutzt das Volume noch | StatefulSet zuerst löschen, `describe pvc` prüfen |
| ArgoCD: `evalsymlink failure ... no such file` | `kustomization.yaml` verweist auf gelöschte Datei (z. B. `pdb.yaml`, `sync/deployment.yaml`) | Eintrag entfernen, `kubectl kustomize manifests/adguard` lokal testen |
| StatefulSet-Sync schlägt fehl (`forbidden`) | `volumeClaimTemplates` geändert | StatefulSet und PVCs von Hand löschen (siehe Migration) |
| Node-Drain hängt | Altes PDB noch im Cluster | `kubectl -n adguard delete pdb adguard` |
| Pod startet nach Node-Ausfall nicht neu | Longhorn Pod Deletion Policy nicht gesetzt | Einstellung auf `delete-statefulset-pod` setzen |
| Keine Queries im Log | Clients nutzen anderen/zweiten DNS, Query-Log deaktiviert | DHCP-DNS prüfen, *Settings → General settings → Query log configuration* |
| Ingress landet in `default`, 503/404 | Manuelles `kubectl apply` ohne Namespace | Objekte in `default` löschen, nur per ArgoCD/Kustomize deployen |
| Zwei Ingresses (`adguard` und `cm-acme-http-solver-*`) | Solver ist der temporäre cert-manager-Ingress | Normal, nichts tun |
| ArgoCD: `authentication required` | Privates Repo ohne Credentials, Token abgelaufen oder Platzhalter-URL | Repo unter *Settings → Repositories* mit Token hinterlegen, `repoURL` exakt prüfen |
| HTTPS liefert **403 Forbidden** aus dem LAN | Hairpin-NAT, Traefik sieht WAN-IP | DNS-Rewrite auf `192.168.7.50` (siehe oben) |
| HTTPS liefert **403** von außen | Gewolltes Verhalten | Nichts tun |
| 403 trotz interner IP | IPv6-Client, Netz fehlt in der Liste oder Client-IP geht verloren | `curl -4`, Allowlist ergänzen, `externalTrafficPolicy: Local` |
| Kein Zertifikat | HTTP-01 scheitert | `kubectl -n adguard get certificate,order,challenge` und `describe challenge` |

Nützliche Diagnose:

```bash
kubectl -n adguard get pods,svc,endpoints,ingress,pvc
kubectl -n adguard logs adguard-0 --tail=50
kubectl -n adguard get certificate,order,challenge
kubectl -n argocd logs deploy/argocd-repo-server --tail=50

# Direkt gegen Traefik testen (umgeht DNS)
curl -I --resolve adguard.kmsystems.ddns.net:443:192.168.7.50 https://adguard.kmsystems.ddns.net

# DNS testen
nslookup example.com 192.168.7.55
```

## Sicherheit

- Admin-UI ist nur im Heimnetz erreichbar. Nach Änderungen von außen testen
  (Handy im Mobilfunk): es muss **403** kommen.
- Port 53 nicht am Router ins Internet weiterleiten (offener Resolver, Missbrauch für
  DNS-Amplification). Nur 80/443 für Traefik werden benötigt.
- Starkes Wizard-Passwort verwenden.
- Config-Backups (`AdGuardHome.yaml`) und Secrets (z. B. Repo-Token) nicht ins Git,
  Secrets mit Sealed Secrets oder SOPS verwalten.

## Checkliste neuer Rollout

1. Repo mit Manifesten pushen, ArgoCD-Application anlegen, Sync.
2. `kubectl -n adguard get pvc` zeigt `work-adguard-0` und `conf-adguard-0` mit `longhorn` und `Bound`.
3. Wizard für `adguard-0` durchlaufen (oder Config wiederherstellen).
4. `kubectl -n adguard get pods` zeigt `1/1`, `get svc adguard-dns` zeigt `192.168.7.55`.
5. `nslookup example.com 192.168.7.55` aus dem LAN liefert eine Antwort.
6. DNS-Rewrite für den Ingress-Namen auf `192.168.7.50` setzen.
7. `https://adguard.kmsystems.ddns.net` im LAN erreichbar, Zertifikat gültig.
8. Von außen testen: 403.
9. Router-DHCP auf DNS `192.168.7.55` umstellen.
10. Longhorn Recurring Job für Backups und Pod Deletion Policy einrichten.
