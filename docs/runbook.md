# Runbook - Troubleshooting (Secure Event Ticketing Platform)

Ovaj dokument opisuje stvarne incidente na koje smo naišli tijekom lokalnog i produkcijskog deploymenta, dijagnostiku i korektivne mjere.

---

## Incident 1: Postgres kontejner u CrashLoopBackOff (lokalno, Podman Compose)

**Simptom:** `podman ps -a` pokazuje da je `postgres` kontejner stalno u statusu "Initialized (starting)" i restarta se u krug. API vraća `{"status":"not-ready"}` na `/readyz`.

**Dijagnostika:**
```bash
podman logs devops-project-app_postgres_1
```
Log je pokazao:
```
ls: /docker-entrypoint-initdb.d/init.sql: Permission denied
```

**Uzrok:** SELinux na RHEL host sustavu blokira pristup datoteci koja se mounta s hosta u rootless kontejner. Standardni bind mount (`:ro`) ne prenosi ispravan SELinux security kontekst na datoteku unutar kontejnera.

**Rješenje:** Dodan SELinux label `:Z` na volume mount u `compose.yaml`:
```yaml
volumes:
  - ./infra/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro,Z
```
Nakon toga:
```bash
podman compose down
podman compose up -d
```

**Validacija:** `podman ps -a` pokazuje `postgres` status "Up X seconds (healthy)".

---

## Incident 2: Postgres pod u CrashLoopBackOff na OpenShiftu (drugačiji uzrok od lokalnog)

**Simptom:** Nakon deploya na OpenShift, `oc get pods -n ticketing` pokazuje `postgres` pod u `CrashLoopBackOff`, dok su svi ostali servisi (redis, api, worker, frontend) uredno "Running".

**Dijagnostika:**
```bash
oc logs deployment/postgres -n ticketing
```
Log je pokazao:
```
chmod: /var/lib/postgresql/data: Operation not permitted
initdb: error: could not change permissions of directory "/var/lib/postgresql/data": Operation not permitted
```

**Uzrok:** OpenShift po defaultu dodjeljuje svakom podu **proizvoljan (random) UID** iz sigurnosnih razloga (restricted SCC), umjesto fiksnog UID-a definiranog u slici. Standardna `postgres:16-alpine` slika očekuje da može mijenjati vlasništvo nad `/var/lib/postgresql/data` direktorijem, što s proizvoljnim UID-om nije dopušteno.

**Rješenje:** Postavljena `PGDATA` varijabla na poddirektorij unutar mountanog volumena (poznat obrazac za rad s Postgres slikama na OpenShiftu):
```yaml
env:
  - name: PGDATA
    value: "/var/lib/postgresql/data/pgdata"
```
Primijenjeno preko ConfigMapa i restart deploymenta:
```bash
oc apply -f k8s/01-configmap.yaml
oc rollout restart deployment/postgres -n ticketing
```

**Validacija:**
```bash
oc get pods -n ticketing
# postgres   1/1   Running   0   <age>
```

---

## Incident 3: Frontend ne može doći do API-ja preko javne rute

**Simptom:** Nakon otvaranja frontend Route-a u pregledniku, dropdown za odabir eventa je prazan, a "Output" polje prikazuje:
```json
{"error": "Failed to initialize page", "details": "NetworkError when attempting to fetch resource."}
```

**Dijagnostika:** Provjera ConfigMapa pokazala je da `API_BASE_URL` pokazuje na internu Kubernetes DNS adresu (`http://api:8080`), koja je dostupna samo unutar klastera, ne i iz preglednika korisnika izvana.

**Uzrok:** Interna Service adresa nije dostupna izvan OpenShift klastera. Za pristup iz preglednika potreban je javni OpenShift Route.

**Rješenje:**
1. Kreiran dodatni Route za `api` servis (`k8s/10-route-api.yaml`)
2. Ažuriran `API_BASE_URL` u ConfigMapu na javnu rutu API-ja
3. Restart frontend deploymenta da povuče novu konfiguraciju:
```bash
oc apply -f k8s/10-route-api.yaml
oc apply -f k8s/01-configmap.yaml
oc rollout restart deployment/frontend -n ticketing
```

**Validacija:** Otvaranje frontend URL-a u pregledniku pokazuje popunjen dropdown eventa; klik na "Purchase" vraća `{"message":"Order queued", "orderId":"..."}`.

---

## Incident 4: Loš/pogrešan container registry pri build/push slika

**Simptom:** `podman push` prema početno pretpostavljenom registry URL-u nije uspijevao / slike se nisu pojavljivale u OpenShift internom registryju.

**Dijagnostika:**
```bash
oc registry info
oc get route -n openshift-image-registry
```
Otkriveno je da je ispravna adresa internog registrija drugačija od one koja je prvo pretpostavljena.

**Uzrok:** OpenShift interni registry ima svoju vlastitu rutu (`default-route-openshift-image-registry.apps.<cluster>`) koju treba eksplicitno dohvatiti, a za pull unutar samog klastera koristi se drugačija, interna ClusterIP adresa (`image-registry.openshift-image-registry.svc:5000`).

**Rješenje:** Korištena javna ruta registrija za `podman push` (izvana), a interna ClusterIP adresa u Kubernetes manifestima (`image:` polje u Deploymentima), jer se pull slika izvršava unutar klastera:
```bash
podman login -u $(oc whoami) -p $(oc whoami -t) \
  default-route-openshift-image-registry.apps.<cluster> --tls-verify=false

podman push default-route-openshift-image-registry.apps.<cluster>/ticketing/ticketing-api:1.0.0 --tls-verify=false
```
U `k8s/06-api.yaml`:
```yaml
image: image-registry.openshift-image-registry.svc:5000/ticketing/ticketing-api:1.0.0
```

**Validacija:**
```bash
oc get imagestream -n ticketing
```
pokazuje sve tri slike (`ticketing-api`, `ticketing-frontend`, `ticketing-worker`) s ispravnim tagovima.

---

## Rolling update i rollback (demonstracija, ne incident)

Za potrebe demonstracije I6 ishoda, namjerno je izveden rolling update na novu verziju API slike, a zatim rollback na prethodnu:

```bash
# Rolling update
oc set image deployment/api api=image-registry.openshift-image-registry.svc:5000/ticketing/ticketing-api:1.0.1 -n ticketing
oc rollout status deployment/api -n ticketing
# → deployment "api" successfully rolled out

# Rollback
oc rollout history deployment/api -n ticketing
oc rollout undo deployment/api -n ticketing
oc rollout status deployment/api -n ticketing
# → deployment "api" successfully rolled out
```

Tijekom oba postupka aplikacija je ostala dostupna (RollingUpdate strategija s `maxUnavailable: 0`), bez prekida rada za korisnike.

---

## Opći troubleshooting checklist

1. `oc get pods -n ticketing -o wide` - status svih podova
2. `oc get events -n ticketing --sort-by=.lastTimestamp` - zadnji eventi na namespaceu
3. `oc logs deployment/<servis> -n ticketing --tail=100` - logovi konkretnog servisa
4. `oc describe pod <pod> -n ticketing` - detaljan status, uključujući readiness/liveness probe rezultate
5. Provjera NetworkPolicy ako je promet neočekivano blokiran između servisa
