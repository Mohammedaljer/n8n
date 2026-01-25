<!-- TOC -->

- [n8n V2.0 Migration](#n8n-v20-migration)
  - [Multi‑Stage Build](#multistage-build)
  - [sed‑Patch: Allowlist für JS \& Pythonon](#sedpatch-allowlist-für-js--pythonon)
  - [n8n-config-patcher](#n8n-config-patcher)
    - [​Warum wird er gebraucht?](#warum-wird-er-gebraucht)
    - [Was wird dafür auf dem Host benötigt?](#was-wird-dafür-auf-dem-host-benötigt)

<!-- /TOC -->

# n8n V2.0 Migration

## Multi‑Stage Build
Die `n8n-main`-Service wird nicht direkt aus `n8nio/n8n:2.4.6` gestartet, sondern mit einem **Multi‑Stage Build** erweitert.  
Stage 1 (`alpine:3.20`) installiert `apk-tools-static`, damit ein statisches `apk.static` verfügbar ist. Stage 2 (`n8nio/n8n:2.4.6`) kopiert nur dieses `apk.static` hinein und installiert damit `unzip`/`zip`, obwohl im n8n‑Image selbst oft kein Paketmanager vorhanden ist.
```yml
dockerfile_inline: |
        FROM alpine:3.20 AS apkstage
        RUN apk add --no-cache apk-tools-static
        FROM n8nio/n8n:2.4.6
        USER root
        COPY --from=apkstage /sbin/apk.static /sbin/apk.static
        RUN mkdir -p /etc/apk && \
            echo "http://dl-cdn.alpinelinux.org/alpine/latest-stable/main" > /etc/apk/repositories && \
            echo "http://dl-cdn.alpinelinux.org/alpine/latest-stable/community" >> /etc/apk/repositories && \
            /sbin/apk.static -U --allow-untrusted --no-cache add unzip zip
        USER node
```

## sed‑Patch: Allowlist für JS & Pythonon
Die Allowlist wird nicht nur über `environment:` gesetzt, sondern zusätzlich in `/etc/n8n-task-runners.json` verankert, weil Task‑Runner Prozesse nur “durchgelassene” Variablen verwenden (Filter über `allowed-env`).  [community.n8n](https://community.n8n.io/t/allow-n8n-runners-stdlib-allow-and-alike-to-override-n8n-task-runners-json-config/239644)  
Dazu kopiert `n8n-config-patcher` die Default‑Datei und patcht sie mit `sed`, um (a) Variablennamen in `allowed-env` aufzunehmen und (b) Werte in der JSON zu setzen.

**Relevante Allowlist‑Variablen (Inhalt per `sed` gesetzt):**
- JavaScript:
  - `NODE_FUNCTION_ALLOW_BUILTIN`: erlaubte Node Built‑ins (z. B. `path`, `crypto`), Komma‑Liste oder `*`.
  - `NODE_FUNCTION_ALLOW_EXTERNAL`: erlaubte externe JS‑Pakete, Komma‑Liste oder `*`.  [docs.n8n](https://docs.n8n.io/hosting/configuration/configuration-examples/modules-in-code-node/)
- Python:
  - `N8N_RUNNERS_STDLIB_ALLOW`: erlaubte Python stdlib‑Module (z. B. `os,pathlib,json`), Komma‑Liste oder `*`.
  - `N8N_RUNNERS_EXTERNAL_ALLOW`: erlaubte externe Python‑Pakete, Komma‑Liste oder `*`.
**Anpassung: “alle Libraries” vs. “nur bestimmte”**  
- `N8N_RUNNERS_EXTERNAL_ALLOW="requests,numpy"` nur diese Pakete sind erlaubt.

- `N8N_RUNNERS_EXTERNAL_ALLOW="*"` alle externen Pakete sind erlaubt (falls im Runner vorhanden).

- `N8N_RUNNERS_EXTERNAL_ALLOW=""` keine externen Pakete sind erlaubt

Die Anpassung erfolgt durch Ersetzen der Werte in den entsprechenden `sed -i 's/.../.../g'` Zeilen.

## n8n-config-patcher 
ist ein Hilfs-Container, der nur dafür da ist, eine gültige Runner‑Konfigurationsdatei auf dem Host zu erzeugen/anzupassen, damit JavaScript/Python‑Imports in Code Nodes gezielt erlaubt werden können. Der Hintergrund ist, dass der Runner‑Launcher Env‑Vars nur dann an Runner-Prozesse weitergibt, wenn sie in der Konfig unter allowed-env stehen, und Werte pro Runner über env-overrides gesetzt werden.
```yml
n8n-config-patcher:
    image: n8nio/runners:2.4.6
    container_name: n8n-config-patcher
    restart: "no"
    user: root
    entrypoint: /bin/sh
```

- Es ist ein „One‑shot“-Container (restart: "no"), der beim Deploy startet, seine Aufgabe erledigt und wieder stoppt.
- Er basiert auf n8nio/runners:<version>, weil genau dort die Default-Konfiguration und das Runner‑Setup enthalten sind.
- Er schreibt eine Datei auf den Host, damit n8n-runners diese Datei später als /etc/n8n-task-runners.json mounten kann.

### ​Warum wird er gebraucht?
**Ohne Patch:**
Viele Allowlist‑Env‑Vars (z.B. `NODE_FUNCTION_ALLOW_BUILTIN, N8N_RUNNERS_STDLIB_ALLOW`) kommen zwar im Container‑Environment an, werden aber vom Launcher u. U. nicht an den Runner weitergegeben, wenn sie nicht in allowed-env stehen.
**Mit Patch:**
allowed-env wird erweitert (damit die Variablen durchgelassen werden).
env-overrides wird gesetzt (damit der Runner wirklich die gewünschten Werte nutzt).

### Was wird dafür auf dem Host benötigt?
Ein leeres Verzeichnis auf dem Host (persistenter Ort)
Beispiel: /home/n8n-data/n8n-config/
Dieses Verzeichnis ist das Ziel für die generierte Datei.
Die Konfig-Datei selbst (wird vom patcher erzeugt)
Der patcher erstellt: /home/n8n-data/n8n-config/n8n-task-runners.json
n8n-runners mountet diese Datei nach: /etc/n8n-task-runners.json:ro


