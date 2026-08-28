# Secure Event Ticketing Platform (Sample DevSecOps Project)

Ovaj repozitorij je referentni uzorak aplikacije za kolegij **Uvod u DevOps - DevSecOps**.
Prikazuje cijeli tok: lokalni razvoj kroz Compose i produkcijski deployment kroz Kubernetes manifeste.

## Arhitektura

- `frontend` - web UI za pregled evenata i kupnju karata
- `api` - REST API za evente, narudzbe i health provjere
- `worker` - pozadinska obrada queue poruka
- `postgres` - trajna pohrana narudzbi
- `redis` - queue/cache sloj

### Zašto kontejneri, a ne virtualne mašine

Aplikacija se sastoji od pet neovisnih servisa s različitim resursnim potrebama i ciklusima nadogradnje (frontend se mijenja često, baza rijetko). Kontejneri su odabrani umjesto klasičnih VM-ova iz sljedećih razloga:

- **Brzina pokretanja i resursna učinkovitost** - kontejner dijeli kernel hosta i podiže se u sekundama, dok VM zahtijeva vlastiti OS i minute za boot. Za pet servisa koji se često restartaju (rolling update) ovo je presudno.
- **Izolacija na razini procesa je dovoljna** - servisi ne trebaju punu hardversku virtualizaciju (odvojen kernel, virtualni disk), samo izolaciju procesa/mreže/filesystema koju daje kontejner.
- **Konzistentnost okruženja** - ista slika (`node:20-alpine`) vrti se identično na developerovom laptopu (Podman Compose) i na OpenShift klasteru, čime se eliminira "radi na mom računalu" problem koji je čest kod VM-ova s ručno konfiguriranim okruženjem.
- **Orkestracija** - Kubernetes/OpenShift upravlja skaliranjem, health checkovima i rolling update/rollback postupkom na razini kontejnera; ekvivalentna automatizacija nad VM-ovima zahtijevala bi puno kompleksniji alat (npr. Terraform + Ansible + load balancer ručno).

Kompromis: kontejneri dijele kernel hosta, pa je izolacija slabija nego kod VM-a (relevantno za multi-tenant scenarije s međusobno nepovjerljivim opterećenjima). Za ovaj projekt, gdje svi servisi pripadaju istoj aplikaciji i istom timu, taj kompromis je prihvatljiv.

### Odabir servisa i njihove uloge

| Servis | Uloga | Zašto zaseban kontejner |
|---|---|---|
| `frontend` | Statički web UI, poziva API iz preglednika | Neovisan lifecycle (UI se mijenja često, ne zahtijeva restart backend logike) |
| `api` | REST sloj - prima narudžbe, stavlja ih u red čekanja | Skalira se neovisno (2 replike), stateless pa je bez problema horizontalno skalabilan |
| `worker` | Asinkrono obrađuje narudžbe iz reda i upisuje u bazu | Odvojen od API-ja da spor upis u bazu ne blokira HTTP odgovore korisniku |
| `postgres` | Trajna pohrana narudžbi | Jedini servis sa stanjem (stateful), zahtijeva perzistentni volume |
| `redis` | Red čekanja između API-ja i workera | Razdvaja primanje zahtjeva od njihove obrade (decoupling), omogućuje da API ostane brz čak i kad je worker zauzet |

### Arhitektura i međuservisna komunikacija

Tok jedne narudžbe kroz sustav:

1. Korisnik u pregledniku šalje zahtjev na `frontend` (port 3000)
2. `frontend` proslijeđuje poziv prema `api` (port 8080) preko interne mreže (Compose network lokalno, Kubernetes Service u produkciji)
3. `api` upisuje narudžbu u Redis red čekanja (port 6379) i odmah vraća `202 Accepted` korisniku - korisnik ne čeka da se narudžba stvarno obradi
4. `worker` u petlji čita iz Redis reda i upisuje obrađenu narudžbu u PostgreSQL (port 5432)
5. Korisnik može naknadno dohvatiti status narudžbe pozivom prema `api`, koji čita direktno iz baze

Ovaj obrazac (asinkrona obrada preko reda čekanja) osigurava da baza podataka nikad nije direktno izložena naglim vršnim opterećenjima s fronte - `api` i `worker` djeluju kao zaštitni sloj.

### Usklađenost pristupa s ciljevima projekta

Cilj projekta bio je demonstrirati siguran, ponovljiv i automatiziran put od lokalnog razvoja do produkcije. Odabrana arhitektura to ispunjava jer: (a) svaki servis ima jasno definiranu, usku odgovornost pa se može neovisno testirati i skalirati, (b) ista definicija servisa (Dockerfile) koristi se lokalno i u produkciji bez izmjena, i (c) asinkrona obrada preko Redisa omogućuje da se demonstrira rolling update API servisa bez gubitka narudžbi koje su već u redu čekanja.

### Brza validacija funkcionalnosti

1. Health API:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

2. Dohvati evente:

```bash
curl http://localhost:8080/events
```

3. Posalji narudzbu:

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
```

4. Provjeri obradene narudzbe:

```bash
curl http://localhost:8080/tickets/orders
```

5. UI:
   - Otvori `http://localhost:3000`

## Sigurnosni elementi

- Multi-stage Docker build i non-root runtime korisnik
- Secret + ConfigMap odvojena konfiguracija
- Liveness/Readiness probe
- Resource requests/limits
- ServiceAccount + RBAC
- NetworkPolicy segmentacija
- Trivy skeniranje slika u CI pipelineu

Detalji skeniranja: `docs/security/image-scan-report.md`

Detaljan runbook za incidente: `docs/runbook.md`
