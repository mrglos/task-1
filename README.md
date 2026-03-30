# Task 1 - GitOps Platform

Niniejsze repozytorium zawiera kompletną implementację platformy GitOps opartej na **Argo CD** oraz **Helm**, służącej do automatycznego wdrażania aplikacji Spring Boot API na klastry Kubernetes.

## 🏗 Architektura i Stack Techniczny

- **Kubernetes:** Lokalny klaster [Kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker).
- **GitOps:** Argo CD (ApplicationSet, AppProject).
- **Package Manager:** Helm (Multiple Sources Pattern).
- **Ingress:** Ingress Nginx Controller.
- **Separacja Konfiguracji:** Kod infrastruktury (Helm Chart) jest odseparowany od konfiguracji środowiskowej (GitOps Config).

## 🚀 Szybki Start (Bootstrap)

### 1. Przygotowanie klastra

Projekt został przygotowany i przetestowany na klastrze `kind`.

```bash
kind create cluster --name dev-cluster
```

### 2. Instalacja Ingress Nginx

Instalacja kontrolera Ingress jest niezbędna do obsługi ruchu HTTP.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### 3. Instalacja Argo CD

Argo CD instalujemy w dedykowanym namespace. Ze względu na duży rozmiar manifestów (CRD), używamy flagi `--server-side`, aby uniknąć błędów limitu adnotacji `kubectl.kubernetes.io/last-applied-configuration`.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side
```

### 4. Dostęp do panelu Argo CD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Hasło administratora (użytkownik: admin):
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 🛠 Wdrożenie Aplikacji

Aby uruchomić system GitOps, wystarczy zaaplikować konfigurację projektu i ApplicationSet:

```bash
kubectl apply -f gitops-config/project.yaml
kubectl apply -f gitops-config/applicationset.yaml
```

## 📝 Decyzje Architektoniczne i Workaroundy

### Symulacja środowiska Multi-Cluster (Workaround)

Zgodnie z wymaganiami zadania, platforma została zaprojektowana pod kątem wdrażania na dwa fizyczne klastry (`dev-global-cluster-0` oraz `prd-global-cluster-5`).

**Na potrzeby lokalnej prezentacji (Kind):**

- Zastosowano mechanizm mapowania klastrów wewnątrz `AppProject`.
- Jako że dysponujemy jednym lokalnym klastrem, obie aplikacje (`dev` i `prd`) są wdrażane na ten sam serwer (`https://kubernetes.default.svc`), ale do **izolowanych namespace'ów**:
  - `spring-boot-api-dev`
  - `spring-boot-api-prd`

Dzięki temu w panelu Argo CD można zaobserwować separację konfiguracji (np. 3 repliki na dev vs 5 na prd) przy użyciu tych samych szablonów Helm.

### Bezpieczeństwo (RBAC)

Zasób `AppProject` został skonfigurowany w modelu **Least Privilege**:

- Ograniczono źródła prawdy tylko do autoryzowanych repozytoriów GitHub (ochrona przed wdrożeniami z nieznanych źródeł).
- Ścisłe mapowanie klastrów docelowych z dozwolonymi namespace'ami zapobiega omyłkowemu nadpisaniu środowisk.
- Zablokowano możliwość tworzenia zasobów o zasięgu klastrowym (Cluster-scoped), z wyjątkiem uprawnień do tworzenia `Namespace`.

### Zarządzanie Sekretami i "Secret Rotation Trap" (Zbieg okoliczności rotacji)

W obecnej implementacji (`secret.yaml`), do generowania losowych haseł (np. `DB_PASSWORD` oraz `API_KEY`) wykorzystano funkcję Helm `randAlphaNum`.

W połączeniu z mechanizmem GitOps (Argo CD) oraz adnotacją `checksum/config` w Deployment, tworzy to zjawisko ciągłych restartów podów (tzw. "Secret Rotation Trap"). Przy każdej synchronizacji Argo CD, Helm generuje nowe wartości sekretów, co zmienia sumę kontrolną w Deployment i wymusza Rolling Update, nawet jeśli kod aplikacji się nie zmienił.

**Rozwiązanie na środowisku produkcyjnym:**
W środowisku komercyjnym wyeliminowano by ten problem, stosując jedno z poniższych rozwiązań:

1. Użycie funkcji Helm `lookup` wewnątrz szablonu, która sprawdza, czy dany Secret już istnieje w klastrze i generuje nowe hasło tylko przy pierwszym wdrożeniu.
2. Zewnętrzne zarządzanie sekretami (np. **External Secrets Operator** integrujący się z AWS Secrets Manager / HashiCorp Vault lub **Sealed Secrets**), co jest rekomendowaną praktyką (Best Practice) w architekturze GitOps, pozwalającą na bezpieczne przechowywanie zaszyfrowanych sekretów w repozytorium.
