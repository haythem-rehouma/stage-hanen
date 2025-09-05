# Énoncé de mission – Automatisation complète du SaaS Wiki.js + SSO + Paiement

## 0) Objectif global

Mettre en place, en Infrastructure-as-Code et Playbooks, une plateforme SaaS composée de :

* **Un SSO unique** (Keycloak) pour l’authentification de tous les wikis.
* **Trois instances Wiki.js** publiques : `ia.`, `devops.`, `cyber.` (une base PostgreSQL par wiki).
* **Un reverse-proxy** (Traefik recommandé ; Nginx possible) avec TLS automatique.
* **Un stockage d’objets** (MinIO/S3) pour les uploads/backup.
* **Un pipeline d’automatisation** qui, après un paiement Stripe, **crée/actualise** l’utilisateur dans Keycloak et **assigne les accès** selon le **plan** (Starter/Pro/Enterprise).
* **Option Enterprise+** : provisionner un **wiki dédié** par client (sous-domaine, DB, service).

> Contraintes : reproductible, idempotent, documenté, secrets chiffrés, logs exploitables, monitoring basique.



## 1) Architecture cible (à livrer)

```mermaid
flowchart LR
  U[Utilisateur] -->|HTTPS| RP[Traefik/Nginx]
  RP -->|/auth| KC[Keycloak (realm: saas)]
  RP --> IA[Wiki.js IA]
  RP --> DV[Wiki.js DevOps]
  RP --> CY[Wiki.js Cyber]
  IA --- PG[(PostgreSQL)]
  DV --- PG
  CY --- PG
  IA --- S3[(MinIO/S3)]
  DV --- S3
  CY --- S3
  STR[Stripe Billing] -->|Webhook| N8N[n8n Flows]
  N8N -->|Keycloak Admin API| KC
  N8N -->|Webhook Infra| OPS[Provisioner/Ansible AWX]
  OPS -->|Compose/Helm| IA
  OPS --> DV
  OPS --> CY
```

**DNS publics (Cloudflare conseillé)**
`app.domaine.com` (portail/marketing), `sso.domaine.com` (Keycloak), `ia.`, `devops.`, `cyber.`, et sous-domaines clients Enterprise (`clientX.domaine.com`).



## 2) Périmètre & choix technos

* **Terraform** (infra)

  * DNS (Cloudflare), VM (provider au choix), sécurité (firewall), option Postgres managé + Object Storage (S3/R2).
* **Ansible** (configuration)

  * Installation Docker + Compose plugin, déploiement Traefik, Keycloak, MinIO, Postgres (si non managé), 3× Wiki.js.
  * Templates `.env` et `docker-compose.yml`.
* **n8n** (automations)

  * Écoute Webhook Stripe → appels Keycloak Admin API (create user, set groups).
  * (Enterprise) déclenchement du provisionnement d’une instance dédiée.
* **Option** Kubernetes (pile Helm) — *hors périmètre initial, seulement si demandé.*



## 3) Décomposition en livrables (obligatoires)

### 3.1 Repo mono « wikisaas-platform »

```
wikisaas-platform/
├─ terraform/
│  ├─ providers.tf
│  ├─ variables.tf
│  ├─ dns.tf
│  ├─ vm.tf
│  ├─ outputs.tf
│  └─ README.md
├─ ansible/
│  ├─ inventories/prod/hosts.ini
│  ├─ group_vars/all.yml
│  ├─ roles/
│  │  ├─ common/ (MAJ, users, firewall)
│  │  ├─ docker/ (engine + compose)
│  │  ├─ traefik/
│  │  │  ├─ files/docker-compose.yml
│  │  │  └─ templates/.env.j2
│  │  ├─ keycloak/
│  │  │  ├─ files/docker-compose.yml
│  │  │  ├─ templates/.env.j2
│  │  │  └─ templates/keycloak-init.sh.j2
│  │  ├─ minio/
│  │  ├─ postgres/
│  │  └─ wiki/ (ia/devops/cyber)
│  │     ├─ files/docker-compose.yml
│  │     └─ templates/.env.j2
│  └─ site.yml
├─ provisioner/        # optionnel si on ne passe pas par AWX
│  └─ server.js        # endpoint /deploy-new-wiki
├─ n8n/
│  └─ flows/stripe-to-keycloak.json
├─ .github/workflows/
│  ├─ terraform.yml
│  └─ ansible-lint.yml
└─ README.md
```

### 3.2 Fichiers/artefacts attendus

* **Terraform** : records DNS, création VM(s), variables d’entrées documentées, outputs (IPs, URLs).
* **Ansible** :

  * Rôle `traefik` : TLS auto (Let’s Encrypt), redirection 80→443, headers sécurité.
  * Rôle `keycloak` : conteneur + script d’initialisation **non-interactif** (realm `saas`, clients `wiki-ia`, `wiki-devops`, `wiki-cyber`, mappers `email, profile, groups`, groupes `STARTER|PRO|ENTERPRISE`).
  * Rôle `wiki` (x3) : conteneurs Wiki.js, DB PostgreSQL nommées `wiki_ia`, `wiki_devops`, `wiki_cyber`.
  * Rôle `minio` (ou liaison S3/R2).
* **n8n** : flow importable (JSON) comportant les nœuds suivants :

  * `Stripe Trigger` → `Function (extract plan + email)` → `HTTP Request (Keycloak: find user)` → `IF (not found ⇒ create)` → `HTTP Request (assign group)` → `IF (plan == Enterprise ⇒ HTTP Request vers /deploy-new-wiki)` → `Email (onboarding)`.
* **Docs** :

  * `RUNBOOK.md` (démarrage/arrêt/rollback, rotation certificats, reset admin Keycloak).
  * `SECURITY.md` (gestion secrets, accès admin, sauvegardes/restauration).
  * `TESTS.md` (tests d’acceptation + scripts).


## 4) Spécifications détaillées

### 4.1 DNS & reverse-proxy

* **Entrées** : `sso`, `ia`, `devops`, `cyber`, `app` → IP publique.
* **TLS** : ACME via Traefik (certresolver `le`).
* **Headers minimum** :

  * `Strict-Transport-Security: max-age=31536000; includeSubDomains`
  * `X-Frame-Options: DENY`
  * `X-Content-Type-Options: nosniff`

### 4.2 Keycloak (realm `saas`)

* **Clients** : `wiki-ia`, `wiki-devops`, `wiki-cyber` (type **Confidential**).
* **Valid Redirect URIs** par wiki :

  * `https://ia.domaine.com/login/oidc/callback`
  * `https://devops.domaine.com/login/oidc/callback`
  * `https://cyber.domaine.com/login/oidc/callback`
* **Web Origins** : l’URL exacte ou `+` (à restreindre ensuite).
* **Mappers** obligatoires : `email`, `preferred_username`, `given_name`, `family_name`, **`groups`** (claim JSON `groups`).
* **Groupes** : `STARTER`, `PRO`, `ENTERPRISE`.
* **Script init automatisé** (via `kcadm.sh` au premier boot) pour : créer realm, clients, mappers, groupes, admin local.

### 4.3 Wiki.js (x3)

* **DB** : PostgreSQL; une base par wiki (`wiki_ia`, `wiki_devops`, `wiki_cyber`).
* **OIDC** :

  * Issuer : `https://sso.domaine.com/realms/saas`
  * ClientID/Secret : propres à chaque wiki
  * Scope : `openid profile email`
  * `Auto-create users = ON`, `Group claim = groups`
  * Mapping groupes→rôles Wiki.js documenté (au moins `reader`, `writer`, `admin`).
* **Uploads** : MinIO/S3 (config storage de Wiki.js).

### 4.4 Plans & autorisations

* **Starter (29,99 €)** : accès **DevOps** uniquement → groupe Keycloak `STARTER`.
* **Professional (99 €)** : accès **IA + DevOps** → `PRO`.
* **Enterprise (299 €)** : **IA + DevOps + Cyber** + utilisateurs illimités → `ENTERPRISE`.
* **Enterprise+ (option)** : wiki **dédié** (`clientX.domaine.com`) → création dynamique d’une **base** + **service** + **client OIDC** + **DNS**.

### 4.5 Stripe → n8n → Keycloak/Provisioning

* **Produits & Prices** : créer 3 produits dans Stripe (Starter/Pro/Enterprise).
* **Webhook** écoutés : `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`.
* **Règles** :

  * À la création/upgrade : trouver/creer l’utilisateur Keycloak (email Stripe), assigner le **groupe** correspondant.
  * À la résiliation/downgrade : ajuster le groupe.
  * **Enterprise** : appeler un endpoint interne `/deploy-new-wiki` avec payload `{customer_email, subdomain}` pour déployer l’instance dédiée (idempotent).
* **Journalisation** : chaque flow n8n écrit un log structuré (date, eventId Stripe, action, résultat).

### 4.6 Sécurité & secrets

* Secrets **jamais** en clair dans Git.
* **Ansible Vault** pour `group_vars/all.vault.yml`.
* Variables Terraform sensibles gérées via TF Cloud ou `*.tfvars` chiffré (hors Git).
* Comptes admin : générer **mots de passe forts** et les stocker dans un coffre (Bitwarden, Vault).

### 4.7 Sauvegardes & restauration

* **PostgreSQL** : `pg_dump` quotidiennes (cron dans un conteneur dédié) vers MinIO/S3 avec rétention (7/30 jours).
* **MinIO** : versioning activé côté bucket.
* **Test de restore** documenté (sandbox).

### 4.8 Observabilité

* **Logs** : rediriger stdout conteneurs → journald ; fournir une commande `journalctl` par service.
* **Monitoring** (option) : dockerized Prometheus + Grafana, au minimum dashboard CPU/RAM, latences Traefik, 4xx/5xx.



## 5) Exemples concrets à livrer (extraits)

### 5.1 `docker-compose.yml` – Traefik (extrait)

```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--providers.docker=true"
      - "--certificatesresolvers.le.acme.tlschallenge=true"
      - "--certificatesresolvers.le.acme.email=${ACME_EMAIL}"
      - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
    ports: ["80:80","443:443"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
```

### 5.2 Keycloak – script d’initialisation (extrait `keycloak-init.sh`)

```bash
/opt/keycloak/bin/kcadm.sh config credentials --server "$KC_URL" --realm master \
  --user "$KC_ADMIN" --password "$KC_ADMIN_PASSWORD"

# realm
/opt/keycloak/bin/kcadm.sh create realms -s realm=saas -s enabled=true

# groupes plans
for g in STARTER PRO ENTERPRISE; do
  /opt/keycloak/bin/kcadm.sh create groups -r saas -s name="$g"
done

# client wiki-ia (confidential)
CID=$(/opt/keycloak/bin/kcadm.sh create clients -r saas -f - <<EOF
{ "clientId":"wiki-ia","protocol":"openid-connect","publicClient":false,
  "redirectUris":["https://ia.${BASE_DOMAIN}/login/oidc/callback"],
  "webOrigins":["https://ia.${BASE_DOMAIN}"],"standardFlowEnabled":true }
EOF
)
```

### 5.3 Wiki.js – `.env` (template Jinja)

```dotenv
WIKI_PORT=3000
DB_TYPE=postgres
DB_HOST={{ pg_host }}
DB_PORT=5432
DB_USER={{ pg_user }}
DB_PASS={{ pg_pass }}
DB_NAME={{ db_name }}
```

### 5.4 n8n – logique du flow (pseudocode)

```
StripeTrigger -> Function(extract email, plan) 
-> HTTP(Keycloak: search user by email)
   -> IF not found -> HTTP(Keycloak: create user + verify)
-> HTTP(Keycloak: assign to group {plan})
-> IF plan == ENTERPRISE -> HTTP(POST /deploy-new-wiki {email, subdomain})
-> Email(Welcome + liens)
```



## 6) Procédure d’installation (Definition of Ready)

1. `terraform init/plan/apply` → obtention IP/hostnames.
2. `ansible-playbook -i inventories/prod site.yml` → stack opérationnelle.
3. Keycloak accessible via `https://sso.domaine.com` (admin créé par script).
4. Les trois wikis répondent en HTTPS et redirigent l’auth vers Keycloak.
5. n8n déployé et webhook Stripe configuré.



## 7) Tests d’acceptation (Definition of Done)

* **SSO**

  * Connexion sur chaque wiki crée automatiquement l’utilisateur depuis Keycloak (groupes reçus).
  * Groupes `STARTER/PRO/ENTERPRISE` mappés vers rôles Wiki.js selon le plan.
* **Stripe→n8n**

  * Paiement test Stripe (mode test) : l’utilisateur reçoit le bon groupe.
  * Annulation/downgrade : ajustement du groupe dans Keycloak (propagé au login suivant).
* **Enterprise+ (option)**

  * Appel de l’endpoint `/deploy-new-wiki` crée une 4ᵉ base, un service Wiki.js, une entrée DNS, et un nouveau **client OIDC** ; connexion réussie.
* **Sécurité**

  * TLS valide, redirection 80→443, headers de sécurité présents.
  * Secrets absents du dépôt Git ; Ansible Vault opérationnel.
* **Ops**

  * Backups PG présents dans MinIO/S3 ; procédure de restore testée.
  * `journalctl` montre les logs des services ; 5xx Traefik < 1% en test de charge basique.



## 8) Bonnes pratiques exigées

* **Idempotence** Ansible (re-lancement sans effets secondaires).
* **Lint** Terraform & Ansible.
* **README** clair pour chaque dossier.
* **Commits** petits et descriptifs.
* **Variables** factorisées ; pas de duplication de config entre wikis (utiliser templates).


## 9) Bonus (si temps disponible)

* Portail `app.domaine.com` (Landing + Checkout Stripe).
* Matomo/PostHog pour analytics produit.
* Grafana + dashboards.
* Script de **rotation de secrets** (Keycloak client secrets, DB passwords).



## 10) Remise attendue

* Lien du repo privé + instructions pour exécuter :

  ```
  cd terraform && terraform apply
  ansible-playbook -i ansible/inventories/prod site.yml
  ```
* Exports : `n8n/flows/stripe-to-keycloak.json`, export Keycloak du realm `saas`.
* Captures d’écran :

  * Login SSO réussi sur les 3 wikis,
  * Utilisateur et groupe dans Keycloak,
  * Stripe → évènement reçu → logs n8n,
  * Backups listés dans MinIO/S3.


