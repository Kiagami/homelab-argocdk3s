# smtp-relay

Einfaches SMTP-Relay (Postfix, Image `boky/postfix`) im Namespace `smtp-relay`.
Dienste im Cluster (Alertmanager, Grafana, Uptime-Tools usw.) senden Mails ohne Login an das Relay.
Das Relay leitet sie authentifiziert und per TLS an den echten Mailanbieter (z. B. Gmail) weiter.

```
apps/smtp-relay.yaml                       # ArgoCD Application
manifests/smtp-relay/deployment.yaml       # Postfix
manifests/smtp-relay/service.yaml          # ClusterIP, Port 25
manifests/smtp-relay/kustomization.yaml
manifests/smtp-relay/secret.example.yaml   # nur Vorlage, nie mit echten Werten committen
```

## Architektur

| Komponente | Datei | Zweck |
|---|---|---|
| Deployment `smtp-relay` | `deployment.yaml` | 1 Pod, Postfix lauscht im Container auf 587 |
| Service `smtp-relay` | `service.yaml` | ClusterIP, Port **25** zeigt auf Container-Port 587 |
| Secret `smtp-relay-credentials` | von Hand angelegt | `username` und `password` für den Mailanbieter |

Funktionsweise:

- Clients im Cluster verbinden sich mit `smtp-relay.smtp-relay.svc.cluster.local:25`, ohne Auth und ohne TLS.
- Postfix nimmt Mails nur aus privaten Netzen an (Pod-Netz von k3s liegt in `10.0.0.0/8`) und nur für die
  Absender-Domains aus `ALLOWED_SENDER_DOMAINS`.
- Von dort geht die Mail an `RELAYHOST` (Anbieter, Port 587, STARTTLS) mit den Zugangsdaten aus dem Secret.
- Der Service ist nur ein ClusterIP. Nicht über MetalLB oder Traefik freigeben, sonst entsteht ein offenes Relay.
- Die Mail-Queue liegt in einem `emptyDir`. Bei einem Pod-Neustart gehen noch nicht zugestellte Mails verloren.
  Für Alerts ist das ok, sonst einen Longhorn-PVC an `/var/spool/postfix` hängen.

## Konfiguration

In `deployment.yaml`:

| Variable | Bedeutung | Beispiel |
|---|---|---|
| `RELAYHOST` | SMTP-Server des Anbieters, in eckigen Klammern (verhindert MX-Lookup) | `[smtp.gmail.com]:587` |
| `ALLOWED_SENDER_DOMAINS` | Absender-Domains, die das Relay annimmt | `gmail.com` |
| `POSTFIX_myhostname` | Hostname, den Postfix nach außen meldet | `smtp-relay.adguard.local` |
| `RELAYHOST_USERNAME` / `RELAYHOST_PASSWORD` | kommen aus dem Secret `smtp-relay-credentials` | |

Die Image-Version in `deployment.yaml` auf einen aktuellen Tag pinnen (Docker Hub: `boky/postfix`).

## Gmail einrichten

Gmail erlaubt kein normales Passwort für SMTP. Du brauchst ein **App-Passwort**.

### 1. App-Passwort erstellen

1. Im Google-Konto die **Bestätigung in zwei Schritten** aktivieren (Voraussetzung für App-Passwörter).
2. Im Google-Konto unter *Sicherheit* nach **App-Passwörter** suchen (oder direkt `myaccount.google.com/apppasswords` öffnen).
3. Einen Namen vergeben, z. B. `k3s-smtp-relay`, und das Passwort erzeugen.
4. Das 16-stellige Passwort kopieren. Es wird nur einmal angezeigt. Leerzeichen weglassen.

Hinweise:

- Menüs und Namen bei Google können sich ändern. Wenn du die Option nicht findest, im Google-Konto nach "App-Passwörter" suchen.
- Bei Google Workspace kann der Admin App-Passwörter deaktiviert haben.
- Das App-Passwort gilt für das ganze Konto. Wenn es verloren geht, im selben Menü widerrufen und neu erstellen.

### 2. Secret anlegen

Das Secret nicht ins Git legen, sondern von Hand erstellen (oder später per Sealed Secrets/SOPS):

```bash
kubectl create namespace smtp-relay
kubectl -n smtp-relay create secret generic smtp-relay-credentials \
  --from-literal=username='deine.adresse@gmail.com' \
  --from-literal=password='abcdefghijklmnop'
```

`username` ist die vollständige Gmail-Adresse, `password` das App-Passwort.

### 3. Deployment anpassen

In `deployment.yaml`:

```yaml
- { name: RELAYHOST, value: "[smtp.gmail.com]:587" }
- { name: ALLOWED_SENDER_DOMAINS, value: "gmail.com" }
```

Falls dein Anbieter TLS erzwingen soll, kann zusätzlich `RELAYHOST_TLS_LEVEL` gesetzt werden
(Variable des `boky/postfix`-Images, Werte siehe dessen Doku).

### 4. Absenderadresse beachten

Gmail schreibt die Absenderadresse (`From`) auf das Konto um, mit dem du dich anmeldest.
Mails kommen also immer von `deine.adresse@gmail.com` an, egal welche `smtp_from` der Client setzt.
Eine andere Absenderadresse geht nur, wenn sie in Gmail als Alias unter *Einstellungen → Konten → Mail senden als* eingetragen und bestätigt ist.

Limits: Ein kostenloses Gmail-Konto darf nur eine begrenzte Zahl von Mails pro Tag senden (aktuell etwa 500 Empfänger). Für Alerts reicht das, bei Alert-Stürmen kann es knapp werden.

### 5. Deployen

1. `repoURL` in `apps/smtp-relay.yaml` anpassen.
2. Änderungen pushen. Die Application `smtp-relay` per Root-App erkennen lassen oder `kubectl apply -f apps/smtp-relay.yaml` ausführen.
3. In ArgoCD synchronisieren.

Ohne das vorher angelegte Secret bleibt der Pod in `CreateContainerConfigError`. Nach dem Anlegen startet er von selbst.

## Benutzung aus dem Cluster

- Host: `smtp-relay.smtp-relay.svc.cluster.local`
- Port: `25`
- Auth: keine, TLS: aus (bleibt im Cluster)
- Absender: bei Gmail die eigene Adresse oder ein bestätigter Alias

Alertmanager:

```yaml
global:
  smtp_smarthost: 'smtp-relay.smtp-relay.svc.cluster.local:25'
  smtp_from: 'deine.adresse@gmail.com'
  smtp_require_tls: false

receivers:
  - name: mail
    email_configs:
      - to: 'empfaenger@example.com'
```

Grafana (`grafana.ini` oder Helm-Values):

```ini
[smtp]
enabled = true
host = smtp-relay.smtp-relay.svc.cluster.local:25
from_address = deine.adresse@gmail.com
```

Aus anderen Namespaces gilt derselbe Hostname. Der Namespace des Absenders spielt keine Rolle.

## Testen

Relay erreichbar?

```bash
kubectl -n smtp-relay get pods
kubectl -n smtp-relay logs deploy/smtp-relay --tail=30
kubectl -n smtp-relay run mailtest --rm -it --image=busybox --restart=Never -- \
  sh -c 'echo QUIT | nc smtp-relay 25'
```

Testmail senden, direkt im Relay-Pod (testet Postfix und die Weiterleitung an den Anbieter):

```bash
kubectl -n smtp-relay exec deploy/smtp-relay -- sh -c \
  'printf "Subject: Relay-Test\n\nHallo aus k3s\n" | sendmail -f deine.adresse@gmail.com empfaenger@example.com'
```

Testmail über den Service senden (testet zusätzlich den Weg aus dem Cluster):

```bash
kubectl -n smtp-relay run mailtest --rm -i --image=busybox --restart=Never -- sh -c '
  { sleep 1; echo "HELO test"; sleep 1;
    echo "MAIL FROM:<deine.adresse@gmail.com>"; sleep 1;
    echo "RCPT TO:<empfaenger@example.com>"; sleep 1;
    echo "DATA"; sleep 1;
    printf "Subject: Relay-Test\r\n\r\nHallo aus k3s\r\n.\r\n"; sleep 2;
    echo "QUIT"; } | nc smtp-relay 25'
```

Hinweis: Die Meldungen `lost connection after CONNECT` im Log kommen von den Readiness- und Liveness-Probes (TCP-Check durch kubelet) und sind normal.

Zustellung prüfen:

```bash
kubectl -n smtp-relay logs deploy/smtp-relay | grep -E "status=(sent|bounced|deferred)"
```

- `status=sent`: Anbieter hat die Mail angenommen. Spam-Ordner des Empfängers prüfen.
- `status=deferred` oder `bounced`: Grund steht in derselben Logzeile.

## Troubleshooting

| Symptom | Ursache | Lösung |
|---|---|---|
| Pod `CreateContainerConfigError` | Secret `smtp-relay-credentials` fehlt oder Key-Namen falsch | Secret mit `username` und `password` im Namespace `smtp-relay` anlegen |
| Log: `SASL authentication failed` / `Username and Password not accepted` | Falsches Passwort, bei Gmail kein App-Passwort, Leerzeichen im Passwort | App-Passwort neu erstellen, ohne Leerzeichen eintragen, Pod neu starten |
| Log: `Relay access denied` | Absender-Domain nicht in `ALLOWED_SENDER_DOMAINS` oder Client-IP nicht im erlaubten Netz | Domain ergänzen, Absenderadresse prüfen |
| Log: `Connection timed out` zum Relayhost | Ausgehender Port 587 blockiert (Firewall/UniFi) oder DNS-Problem | Egress-Regel für 587 prüfen, `nslookup smtp.gmail.com` aus einem Pod testen |
| Mail kommt an, aber mit falschem Absender | Gmail schreibt `From` auf das Konto um | Alias in Gmail einrichten oder die Gmail-Adresse als Absender nutzen |
| Mail kommt nicht an, Log zeigt `sent` | Landet im Spam beim Empfänger | Spam-Ordner prüfen, Absenderadresse eines Alerts als "kein Spam" markieren |
| Nach Pod-Neustart fehlen Mails | Queue liegt im `emptyDir` | Longhorn-PVC an `/var/spool/postfix` hängen |
| Passwort geändert, Relay nutzt altes | Env aus Secret wird nur beim Start gelesen | `kubectl -n smtp-relay rollout restart deploy/smtp-relay` |

## Sicherheit

- Service nur als ClusterIP betreiben, Port 25 nicht über den Router oder MetalLB freigeben.
- Zugangsdaten nicht im Klartext ins Git. Für GitOps später Sealed Secrets oder SOPS nutzen.
- Für Alerts ein eigenes Gmail-Konto verwenden, nicht das Hauptkonto.
- Optional: eine NetworkPolicy, die Zugriff auf Port 587 nur aus den Namespaces zulässt, die Mails senden sollen.
