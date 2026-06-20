# Secure Event Ticketing Platform (Sample DevSecOps Project)

Ovaj repozitorij je referentni uzorak aplikacije za kolegij **Uvod u DevOps - DevSecOps**.
Prikazuje cijeli tok: lokalni razvoj kroz Compose i produkcijski deployment kroz Kubernetes manifeste.

## Arhitektura

- `frontend` - web UI za pregled evenata i kupnju karata (port 3000)
- `api` - REST API za evente, narudžbe i health provjere (port 8080)
- `worker` - pozadinska obrada queue poruka (bez izloženog porta)
- `postgres` - trajna pohrana narudžbi
- `redis` - queue/cache sloj

Tok podataka: `frontend` → `api` → Redis queue → `worker` → Postgres.

## Preduvjeti

- [Podman](https://podman.io/) i `podman compose` (ili `docker-compose` kompatibilan plugin)
- Slobodni portovi na hostu: `3000`, `8080`, `5432`, `6379`

## Lokalni razvoj (1. dio projekta)

### Pokretanje

1. Kopiraj predložak environment varijabli:
   ```bash
   cp .env.example .env
   ```
2. Podigni cijeli stack (build + start svih 5 servisa):
   ```bash
   podman compose up --build -d
   ```
3. Provjeri da su svi servisi gore i da im uptime raste (ne smije stalno pisati "Less than a second" - to bi značilo crash loop):
   ```bash
   podman compose ps
   ```

### Gašenje

```bash
podman compose down
```

Ovo zaustavlja i uklanja kontejnere, ali **ne briše** Postgres podatke (čuvaju se u named volumeu `pgdata`). Ako želiš potpuno čisto stanje (uključujući bazu), koristi `podman compose down -v` — oprezno, ovo nepovratno briše sve podatke.

### Hot-reload za razvoj

Svaki servis (`api`, `worker`, `frontend`) gradi se u `dev` Containerfile stageu koji koristi `nodemon` i bind-montira lokalni `src/` direktorij u kontejner — izmjene koda na hostu odmah se reflektiraju bez ručnog rebuilda.

> **Napomena:** `nodemon` je pokrenut s `--legacy-watch` (polling način rada umjesto `inotify`). Ovo je nužno jer native file-watching ne radi pouzdano preko bind-mounta u (rootless) Podman kontejneru — bez ove zastavice `nodemon` puca s `EMFILE: too many open files` i ulazi u crash loop. Cijena je do ~1s kašnjenja u detekciji promjene, što je za razvoj zanemarivo.

### Validacija funkcionalnosti

Health i readiness provjere:
```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```
`/readyz` provjerava i konekciju na Postgres i na Redis.

Dohvati dostupne evente:
```bash
curl http://localhost:8080/events
```

Pošalji narudžbu (završava u Redis queueu, `worker` je asinkrono obrađuje):
```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
```

Provjeri da je narudžba obrađena (status `"processed"` znači da je `worker` uspješno upisao zapis u Postgres):
```bash
curl http://localhost:8080/tickets/orders
```

UI: otvori `http://localhost:3000` u pregledniku.

### Troubleshooting

| Simptom | Mogući uzrok | Rješenje |
|---|---|---|
| `podman compose up` javlja grešku o spajanju na `podman.sock` | Podman socket servis nije aktivan | `systemctl --user enable --now podman.socket` |
| `api`/`worker` u `podman compose ps` stalno pokazuju "Up Less than a second" | Crash loop, pogledaj `podman compose logs <servis>` | Provjeri da `Dockerfile` ima ispravan `dev` stage i da `package.json` koristi `nodemon --legacy-watch` |
| `curl` na `:8080` vraća "Connection reset by peer" | Kontejner se još pokreće ili je u crash loopu | `podman compose ps` i `podman compose logs api` |
| Postgres podaci nestali nakon `down` | Korišten `down -v` | Izbjegavaj `-v` osim ako namjerno želiš resetirati bazu |

## Sigurnosni elementi

- Multi-stage Docker build i non-root runtime korisnik 
- Trivy skeniranje slika u CI pipelineu — u izradi
- Secret + ConfigMap odvojena konfiguracija
- Liveness/Readiness probe 
- Resource requests/limits
- ServiceAccount + RBAC 
- NetworkPolicy segmentacija 