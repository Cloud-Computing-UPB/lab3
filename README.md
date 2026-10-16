# Cloud Computing – Laboratorul 3

Acest repository continuă aplicația **Ticketing API** din Laboratoarele 1 și 2
și introduce configurarea, stocarea persistentă și reziliența în Kubernetes.
API-ul rulează ca Deployment, iar PostgreSQL rulează ca StatefulSet și folosește
un PersistentVolumeClaim.

## Obiective

La finalul laboratorului veți putea:

- să separați configurația nesensibilă de credențiale prin ConfigMap și Secret
- să explicați diferența dintre Deployment și StatefulSet
- să folosiți un PersistentVolumeClaim creat din `volumeClaimTemplates`
- să configurați startup, liveness și readiness probes
- să definiți requests, limits și un security context
- să observați indisponibilitatea temporară și recuperarea unei dependențe
- să verificați că datele supraviețuiesc recreării Pod-ului PostgreSQL.

## Structura repository-ului

```text
.
├── .github/workflows/ci.yml
├── Dockerfile
├── docker-compose.yml
├── kind-config.yaml
├── k8s
│   ├── 00-namespace.yaml
│   ├── 10-configmap.yaml
│   ├── 11-secret.yaml
│   ├── 20-postgres-service.yaml
│   ├── 21-postgres-statefulset.yaml
│   ├── 30-api-deployment.yaml
│   └── 31-api-service.yaml
├── internal
└── main.go
```

## Cerințe

- Docker;
- `kubectl`
- `kind`
- `curl` sau un client HTTP echivalent.

## Pornire rapidă

Manifestele folosesc imaginea publică
`ghcr.io/cloud-computing-upb/lab3:1.0.0`.

```bash
kind create cluster --name cc-lab3 --config kind-config.yaml

kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/10-configmap.yaml
kubectl apply -f k8s/11-secret.yaml
kubectl apply -f k8s/20-postgres-service.yaml
kubectl apply -f k8s/21-postgres-statefulset.yaml

kubectl rollout status statefulset/postgres -n cc-lab3

kubectl apply -f k8s/30-api-deployment.yaml
kubectl apply -f k8s/31-api-service.yaml

kubectl rollout status deployment/ticketing-api -n cc-lab3
kubectl get all,pvc -n cc-lab3
```

Expuneți local API-ul:

```bash
kubectl port-forward service/ticketing-api -n cc-lab3 8080:80
```

Într-un alt terminal:

```bash
curl http://localhost:8080/livez
curl http://localhost:8080/readyz
curl http://localhost:8080/info
```

## Endpoint-uri

| Metodă | Cale | Descriere |
| --- | --- | --- |
| `GET` | `/health` | Verificare simplă, păstrată pentru compatibilitate |
| `GET` | `/livez` | Confirmă că procesul API rulează |
| `GET` | `/readyz` | Verifică dacă API-ul poate comunica cu PostgreSQL |
| `GET` | `/info` | Versiunea aplicației și Pod-ul care a răspuns |
| `GET` | `/tickets` | Listează tichetele |
| `POST` | `/tickets` | Creează un tichet |
| `GET` | `/tickets/{id}` | Returnează un tichet |
| `PUT` | `/tickets/{id}` | Înlocuiește un tichet |
| `DELETE` | `/tickets/{id}` | Șterge un tichet |

## Configurare

Aplicația acceptă în continuare `DATABASE_URL`. Dacă aceasta nu este definită,
construiește conexiunea din variabilele următoare:

| Variabilă | Sursă în Kubernetes | Descriere |
| --- | --- | --- |
| `APP_VERSION` | ConfigMap | Versiunea afișată de `/info` |
| `PORT` | ConfigMap | Portul HTTP al aplicației |
| `USE_POSTGRES` | ConfigMap | Activează implementarea PostgreSQL |
| `DB_HOST` | ConfigMap | Numele Service-ului PostgreSQL |
| `DB_PORT` | ConfigMap | Portul PostgreSQL |
| `DB_NAME` | ConfigMap | Numele bazei de date |
| `DB_USER` | ConfigMap | Utilizatorul bazei de date |
| `DB_PASSWORD` | Secret | Parola bazei de date |

Valoarea din `k8s/11-secret.yaml` este exclusiv pentru laborator. Un Secret
Kubernetes nu trebuie tratat ca un manager complet de secrete și nu trebuie
să conțină credențiale reale într-un repository public.

## Verificarea persistenței

Creați un tichet:

```bash
curl -X POST http://localhost:8080/tickets \
  -H 'Content-Type: application/json' \
  -d '{"title":"Date persistente","priority":"high","reporter":"student"}'
```

Ștergeți Pod-ul PostgreSQL și urmăriți recrearea acestuia:

```bash
kubectl delete pod postgres-0 -n cc-lab3
kubectl get pods -n cc-lab3 -w
```

După ce PostgreSQL și API-ul redevin Ready, verificați datele:

```bash
curl http://localhost:8080/readyz
curl http://localhost:8080/tickets
```

Tichetul trebuie să existe în continuare deoarece noul Pod reutilizează același
PersistentVolumeClaim.

## Rulare locală fără Kubernetes

```bash
cp .env.example .env
docker compose up --build
```

## Teste

```bash
go test -race ./...
```

## Curățare

```bash
kind delete cluster --name cc-lab3
```

Ștergerea simplă a StatefulSet-ului nu șterge automat PVC-ul. Ștergerea
clusterului local elimină toate resursele create pentru laborator.
