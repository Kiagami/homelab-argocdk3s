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


AdGuard Home läuft als StatefulSet mit 3 Replicas im Namespace `adguard`.
Die Admin-UI ist nur aus dem Heimnetz erreichbar (Traefik-Allowlist, TLS per cert-manager).
DNS läuft über MetalLB, nicht über Traefik.

## Architektur

| Komponente | Datei | Zweck |
|---|---|---|
| StatefulSet `adguard` | `statefulset.yaml` | 3 Pods, je eigene PVCs (`work`, `conf`, local-path) |
| Service `adguard-headless` | `service-ui.yaml` | Stabile Pod-DNS-Namen für den Sync |
| Service `adguard-ui` | `service-ui.yaml` | Web-UI, zeigt nur auf Pod 0 (Origin) |
| Service `adguard-dns` | `service-dns.yaml` | LoadBalancer via MetalLB, `192.168.7.55`, Port 53 UDP/TCP |
| Ingress `adguard` | `ingress.yaml` | `adguard.kmsystems.ddns.net`, TLS, Allowlist-Middleware |
| PodDisruptionBudget | `pdb.yaml` | `minAvailable: 2` |
| Kustomization | `kustomization.yaml` | Setzt `namespace: adguard` für alle Ressourcen |

Wichtige IPs:

- `192.168.7.55`: DNS-Server für alle Clients (MetalLB, `adguard-dns`)
- `192.168.7.50`: Traefik-LoadBalancer (Ziel für interne Auflösung der Web-Namen)

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
    path: adguard
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

## Ersteinrichtung (Setup-Wizard)

Vor dem Wizard lauscht AdGuard nur auf Port **3000**. Die `readinessProbe` prüft TCP 53,
deshalb sind die Pods bis zum Abschluss des Wizards `0/1 Ready`. Das ist erwartet.
Services haben in dieser Zeit keine Endpoints, Ingress und DNS funktionieren noch nicht.

Jeder Pod hat eigene PVCs und muss einzeln eingerichtet werden.

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
- Gleichen Benutzer und gleiches Passwort für alle 3 Pods setzen (wird für den Sync gebraucht)
- Die Seite "Configure your devices" ist nur eine Info-Seite (zeigt Pod-IPs), einfach **Next** und **Open Dashboard**

Danach dasselbe mit `adguard-1` und `adguard-2`. Nach dem Wizard schließt sich Port 3000 im Pod.
`connection refused` auf 3000 heißt also: Pod ist bereits eingerichtet.

Prüfen, welcher Pod noch offen ist: `0/1` = Wizard fehlt, `1/1` = fertig.

```bash
kubectl -n adguard get pods -o wide
kubectl -n adguard get endpoints
kubectl -n adguard get svc adguard-dns   # EXTERNAL-IP = 192.168.7.55
```

Zugriff auf die UI danach über den Ingress oder per Port-Forward auf Port 80:

```bash
kubectl -n adguard port-forward pod/adguard-0 8080:80
```

Hinweis: `kubectl port-forward` braucht keine Pod- oder Node-IP. Es läuft über den API-Server
aus der kubeconfig. Pods auf anderen Nodes sind trotzdem erreichbar.

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
- Router/DHCP verteilt `192.168.7.55` als DNS-Server, ohne externen Fallback-DNS,
  sonst umgehen Geräte AdGuard und den Rewrite
- Kein dauerhafter `/etc/hosts`-Eintrag auf Clients (verdeckt später Fehler)

## Troubleshooting

| Symptom | Ursache | Lösung |
|---|---|---|
| Pods `0/1 Ready`, Services ohne Endpoints | Wizard nicht durchlaufen, Probe prüft Port 53 | Wizard pro Pod per Port-Forward auf 3000 abschließen |
| Portforward auf Nodes landet im Nichts | Port 3000 ist in keinem Service veröffentlicht | `kubectl port-forward pod/adguard-N 3000:3000` nutzen |
| `connection refused` beim Port-Forward auf 3000 | Pod ist bereits eingerichtet (3000 geschlossen) oder falscher Pod | Nächsten Pod nehmen (`adguard-1`, `-2`), UI dann auf Port 80 |
| `localhost:3000` im Browser geht nicht | kubectl läuft auf dem Node, nicht auf dem Notebook | SSH-Tunnel `ssh -L 3000:localhost:3000 root@<node>` |
| `adguard-dns` EXTERNAL-IP `<pending>` | Kein LoadBalancer-Provider oder IP aus dem Pool nicht verfügbar | MetalLB-Pool und Annotation `metallb.universe.tf/loadBalancerIPs` prüfen |
| Ingress landet in `default`, 503/404 | Manuelles `kubectl apply` ohne Namespace | Objekte in `default` löschen, nur per ArgoCD/Kustomize deployen |
| Zwei Ingresses (`adguard` und `cm-acme-http-solver-*`) | Solver ist der temporäre cert-manager-Ingress | Normal, nichts tun |
| ArgoCD: `authentication required` | Privates Repo ohne Credentials, Token abgelaufen oder Platzhalter-URL | Repo unter *Settings → Repositories* mit Token hinterlegen, `repoURL` exakt prüfen |
| HTTPS liefert **403 Forbidden** aus dem LAN | Hairpin-NAT, Traefik sieht WAN-IP | DNS-Rewrite auf `192.168.7.50` (siehe oben) |
| HTTPS liefert **403** von außen | Gewolltes Verhalten | Nichts tun |
| 403 trotz interner IP | IPv6-Client, Netz fehlt in der Liste oder Client-IP geht verloren | `curl -4`, Allowlist ergänzen, `externalTrafficPolicy: Local` |
| Kein Zertifikat | HTTP-01 scheitert | `kubectl -n adguard get certificate,order,challenge` und `describe challenge` |

Nützliche Diagnose:

```bash
kubectl -n adguard get pods,svc,endpoints,ingress
kubectl -n adguard get certificate,order,challenge
kubectl -n argocd logs deploy/argocd-repo-server --tail=50

# Direkt gegen Traefik testen (umgeht DNS)
curl -I --resolve adguard.kmsystems.ddns.net:443:192.168.7.50 https://adguard.kmsystems.ddns.net
```

## Sicherheit

- Admin-UI ist nur im Heimnetz erreichbar. Nach Änderungen von außen testen
  (Handy im Mobilfunk): es muss **403** kommen.
- Port 53 nicht am Router ins Internet weiterleiten (offener Resolver, Missbrauch für
  DNS-Amplification). Nur 80/443 für Traefik werden benötigt.
- Starkes Wizard-Passwort verwenden.
- Secrets (z. B. `adguard-sync-credentials`, Repo-Token) nicht im Klartext ins Git,
  sondern mit Sealed Secrets oder SOPS.

## Sync der Replicas (optional)

`sync/deployment.yaml` in der `kustomization.yaml` erst aktivieren, wenn

1. alle 3 Pods den Setup-Wizard durchlaufen haben und
2. das Secret `adguard-sync-credentials` existiert.

Die Web-UI zeigt bewusst nur auf Pod 0 (Origin). Die Konfiguration wird von dort auf die
anderen Replicas synchronisiert.

## Checkliste neuer Rollout

1. Repo mit Manifesten pushen, ArgoCD-Application anlegen, Sync.
2. Wizard für `adguard-0`, `-1`, `-2` durchlaufen.
3. `kubectl -n adguard get pods` zeigt `1/1`, `get svc adguard-dns` zeigt `192.168.7.55`.
4. `nslookup example.com 192.168.7.55` aus dem LAN liefert eine Antwort.
5. DNS-Rewrite für den Ingress-Namen auf `192.168.7.50` setzen.
6. `https://adguard.kmsystems.ddns.net` im LAN erreichbar, Zertifikat gültig.
7. Von außen testen: 403.
8. Router-DHCP auf DNS `192.168.7.55` umstellen.
