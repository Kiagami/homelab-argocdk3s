# Tailscale Kubernetes Operator & Homelab Subnet Router (Argo CD)

Dieses Deployment installiert den offiziellen **Tailscale Kubernetes Operator** via Argo CD (`apps/timescale-app.yaml`) und stellt über die `Connector`-CRD (`manifests/timescale/connector.yaml`) einen **Subnet Router** für das lokale Homelab-Netzwerk (`192.168.2.0/24`) bereit. Dadurch ist das Homelab auch hinter einem DS-Lite-/CGNAT-Anschluss ohne Portfreigaben erreichbar.

## Verzeichnisstruktur

```text
├── apps/
│   └── timescale-app.yaml
└── manifests/
    └── timescale/
        ├── README.md
        ├── namespace.yaml
        └── connector.yaml
```

---

## 1. Voraussetzungen in Tailscale

### A) ACL-Tags und Auto-Approver konfigurieren
In der [Tailscale Admin Console](https://login.tailscale.com/admin/acls) müssen `tagOwners` (und optional `autoApprovers` für automatische Routen-Freigabe) in der Policy hinterlegt sein:

```json
{
  "tagOwners": {
    "tag:k8s-operator": [],
    "tag:k8s": ["tag:k8s-operator"]
  },
  "autoApprovers": {
    "routes": {
      "192.168.2.0/24": ["tag:k8s"]
    }
  }
}
```

### B) OAuth-Client erstellen
1. In der Tailscale Admin Console zu **Settings → OAuth clients → Generate OAuth client** navigieren.
2. Folgende Scopes auswählen:
   - **Devices → Core:** `Write`
   - **Auth Keys:** `Write`
3. Den Tag **`tag:k8s-operator`** zuweisen und den Client generieren.
4. `Client ID` und `Client Secret` notieren.

---

## 2. Kubernetes-Secret anlegen (einmalig vor dem Sync)

Das Helm-Chart erwartet die OAuth-Zugangsdaten im Secret `operator-oauth` im Namespace `tailscale` (nicht im Git-Repo speichern):

```bash
kubectl create namespace tailscale --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic operator-oauth \
  --namespace tailscale \
  --from-literal=client_id="<DEINE_OAUTH_CLIENT_ID>" \
  --from-literal=client_secret="<DEIN_CLIENT_SECRET>"
```

*(Hinweis: Wenn du SealedSecrets oder ExternalSecrets nutzt, lege `operator-oauth` entsprechend deklarativ darüber an.)*

---

## 3. Argo CD Deployment ausrollen

1. Alle Dateien unter `apps/timescale-app.yaml` und `manifests/timescale/` in das GitOps-Repository pushen.
2. Falls `apps/` bereits von deinem App-of-Apps-Setup überwacht wird, erkennt Argo CD die Application automatisch. Andernfalls manuell anwenden:
   ```bash
   kubectl apply -f apps/timescale-app.yaml
   ```

---

## 4. Verifizierung & Betrieb

1. **Pods prüfen:**
   ```bash
   kubectl get pods -n tailscale
   ```
   Es sollten sowohl der `operator`-Pod als auch der Proxy-Pod (`ts-homelab-subnet-router-...`) im Status `Running` sein.

2. **Connector-Status prüfen:**
   ```bash
   kubectl get connector -n tailscale
   ```

3. **Tailscale Admin Console prüfen:**
   Unter **Machines** erscheinen `k8s-operator` und `k8s-homelab-router` mit der freigegebenen Route `192.168.2.0/24`.

4. **Client-Test (unterwegs / außerhalb des LANs):**
   ```bash
   tailscale status
   ping -c 3 192.168.2.200
   ```

---

## 5. Troubleshooting

- **Connector-CRD fehlt beim ersten Sync:** Falls Argo CD beim ersten Durchlauf meldet, dass `Connector` noch nicht bekannt ist, sorgt `SelfHeal` automatisch für einen erneuten Sync, sobald das Helm-Chart die CRDs installiert hat.
- **Route taucht nicht auf:** Prüfen, ob `tag:k8s` unter `tagOwners` in der Tailscale ACL-Policy eingetragen ist und die Logs des Operators kontrollieren:
  ```bash
  kubectl logs -n tailscale -l app=operator
  ```
- **Kein Routing ins LAN trotz aktivem Tunnel:** Sicherstellen, dass auf den Worker-Nodes IP-Forwarding (`net.ipv4.ip_forward = 1`) aktiv ist und keine lokale Firewall (`iptables`/`nftables`) Forward-Pakete zwischen `tailscale0` und dem physischen Interface verwirft.