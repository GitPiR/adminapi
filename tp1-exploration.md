# TP1 — Exploration

## Mise en place

- Dépôt `adminapi/` créé, fichiers du dossier `projet/` copiés (`docker-compose.yml`,
  `verifier.py`, `contrat-openapi.yaml`, `.env.example`, `surveillant.py`, `sujet.html`,
  `AGENTS.md`, `CLAUDE.md`).
- `.gitignore` créé avant le premier commit (`.env`, `.venv/`, `__pycache__/`, `inventaire.json`).
- `docker compose up -d` : les 4 services démarrent (`docker-api`, `netbox`, `netbox-postgres`,
  `netbox-redis`). NetBox passe `healthy` après quelques minutes.

## 1. Deux conteneurs, l'un étiqueté, l'autre non

```bash
docker run -d --name web1 --label adminapi.managed=true nginx:alpine
docker run -d --name intrus nginx:alpine
```

Les deux tournent (`docker ps`).

## 2. API Docker en cURL

```bash
D=http://127.0.0.1:2375
curl -s $D/version | python3 -m json.tool
```

→ `200`, version de l'API Docker (`ApiVersion: 1.56`).

```bash
curl -s "$D/containers/json?all=true" | python3 -m json.tool
```

→ `200`, liste de **tous** les conteneurs de la machine (y compris ceux de compose :
netbox, postgres, redis, docker-api). Seul `web1` porte `"adminapi.managed": "true"`
dans son champ `Labels` ; `intrus` n'a pas ce label.

```bash
curl -s --get $D/containers/json --data-urlencode 'all=true' \
     --data-urlencode 'filters={"label":["adminapi.managed=true"]}' | python3 -m json.tool
```

→ `200`, liste réduite au seul `web1`.

```bash
curl -s $D/containers/web1/json | python3 -m json.tool
```

→ `200`, détail de `web1`. Le label se trouve ici sous `Config.Labels`, pas à la racine
comme dans la liste — différence de structure entre `/containers/json` (liste) et
`/containers/{id}/json` (détail). `Config.Labels` peut valoir `None` si le conteneur n'a
aucun label.

## 3. Arrêter deux fois, puis demander un conteneur inconnu

```bash
curl -i -X POST $D/containers/web1/stop
```

→ **204 No Content** (arrêt réussi).

```bash
curl -i -X POST $D/containers/web1/stop
```

→ **304 Not Modified** (déjà arrêté — Docker ne renvoie pas d'erreur).

```bash
curl -i $D/containers/n-existe-pas/json
```

→ **404 Not Found**, `{"message":"No such container: n-existe-pas"}`.

**Remarque** : l'API Docker brute répond 204/304/404. Le contrat de notre propre API
(`sujet.html`) demande lui un **409** pour un arrêt sur un conteneur déjà arrêté : il
faudra donc traduire le 304 de Docker en 409 côté API.

## 4. Token NetBox

Interface : http://127.0.0.1:8080 (login `admin` / `admin`).
Documentation Swagger : http://127.0.0.1:8080/api/schema/swagger-ui/

```bash
N=http://127.0.0.1:8080/api
curl -s -X POST $N/users/tokens/provision/ \
     -H "Content-Type: application/json" -H "Accept: application/json" \
     -d '{"username": "admin", "password": "admin"}'
```

→ `200`, réponse contenant `key` et `token`. Token complet reconstruit sous la forme
`nbt_<key>.<token>`, stocké dans `.env` (`NETBOX_TOKEN=`), jamais commité.

## 5. Type de cluster et cluster NetBox

```bash
export NETBOX_TOKEN="nbt_<key>.<token>"
A="Authorization: Bearer $NETBOX_TOKEN"

curl -s -X POST $N/virtualization/cluster-types/ -H "$A" \
     -H "Content-Type: application/json" -H "Accept: application/json" \
     -d '{"name": "Docker", "slug": "docker"}'
```

→ `201`, type de cluster `Docker` créé, `id: 1`.

```bash
curl -s -X POST $N/virtualization/clusters/ -H "$A" \
     -H "Content-Type: application/json" -H "Accept: application/json" \
     -d '{"name": "docker-local", "type": 1}'
```

→ `201`, cluster `docker-local` créé, statut `active`, rattaché au type `Docker` (`id: 1`).
Visible dans NetBox sous **Virtualization → Clusters**.

## 6. Dépôt GitHub

```bash
git add -A
git status                 # .env absent de la liste
git commit -m "Met en place l'environnement du projet"
git branch -M main
git remote add origin https://github.com/<mon-compte>/adminapi.git
git push -u origin main
```

URL du dépôt : _à compléter_
