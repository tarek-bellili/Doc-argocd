# ArgoCD (GitOps continuous delivery)

Ce composant installe ArgoCD (`argoproj/argo-helm`) et l'expose derrière la `Gateway` partagée (`platform-gateway`) livrée par le composant Gateway API/NGF.

## Architecture

Mêmes trois rôles que pour le reste de la plateforme :

- **VM Mirror** — héberge le registre Docker (images) et le dépôt Helm (charts `.tgz`). Une source, rien ne s'y exécute.
- **VM PIC** — lance les playbooks Ansible, pilote le cluster via `kubectl`/`helm`. Contient `bundle/` + `deploy-argocd/`.
- **Cluster K8s** — exécute les workloads, doit pouvoir joindre le Mirror pour tirer images et charts.

**Point important — CRDs embarquées dans `templates/`, pas dans `crds/`** : contrairement à la convention standard (`<chart>/crds/`, jamais mise à jour par `helm upgrade`), le chart argo-cd place ses 3 CRDs (`applications`, `applicationsets`, `appprojects`) dans `templates/crds/` — un choix délibéré des mainteneurs pour qu'elles suivent le cycle de vie normal du chart. Conséquence pratique : `helm show crds` ne retourne **rien** pour ce chart (il ne lit que le dossier `crds/` racine) ; il faut extraire les CRDs via `helm template --show-only templates/crds/<fichier>.yaml` pour obtenir du YAML pur, pas `helm show crds`. Elles sont malgré tout gérées dans un `playbook-crds.yml` séparé plutôt que laissées à Helm (`crds.install: false`), pour garder le contrôle explicite du moment de mise à jour et éviter qu'un `helm uninstall` accidentel les supprime.

**Piège déjà rencontré — boucle de redirection HTTP 307 infinie** : par défaut, `argocd-server` termine son propre TLS en interne (log au démarrage : `tls: true`) et redirige systématiquement vers HTTPS. Derrière une Gateway qui termine déjà le TLS et transmet en clair au backend, ça produit une boucle `307` ininterrompue (`curl -v` montre la même requête répétée indéfiniment). Le correctif n'est **pas** une clé `server.insecure` dans les values (elle n'existe pas dans ce chart — vérifié directement dans `values.yaml`, aucune occurrence hors commentaires sans rapport) mais un argument CLI explicite : `server.extraArgs: ["--insecure"]`, qui indique au binaire qu'un proxy amont gère déjà le TLS.

**Piège déjà rencontré — redirection vers `argocd.example.com`** : sans `global.domain` explicitement renseigné, ArgoCD garde l'URL par défaut du chart (`argocd.example.com`) comme identité interne et y redirige certains flux, quel que soit le hostname réellement utilisé par le `HTTPRoute`. `global.domain: "{{ argocd_hostname }}"` est obligatoire, pas cosmétique — à combiner avec `server.extraArgs: ["--insecure"]` ci-dessus (les deux sont nécessaires, ni l'un ni l'autre seul ne suffit).

**Piège déjà rencontré — ServiceMonitor invisible malgré `enabled: true`** : même mécanisme que pour Harbor. Les 4 `ServiceMonitor` générés par défaut (controller, applicationset-controller, repo-server, server) portent `release: argocd` (le nom de la release Helm), alors que le `serviceMonitorSelector` du CR `Prometheus` attend `release: kube-prometheus-stack`. Chaque bloc `metrics.serviceMonitor` expose `additionalLabels`, utilisé dans le template pour forcer la bonne valeur. **Toujours vérifier après déploiement** — ne jamais se fier au flag `enabled: true` seul.

**Pas de stockage persistant** : ArgoCD (mode non-HA comme en HA via `redis-ha`) ne crée aucun `PersistentVolumeClaim` — `kubectl get pvc -n argocd` reste vide. Le futur passage du cluster de test vers une cible avec Rook-Ceph n'a donc aucun impact sur ce composant.

Le `Certificate argocd-tls` est géré ailleurs, centralisé dans `gateway-system` avec ceux de Harbor et Grafana.

---

## 1️⃣ Déploiement

Depuis la VM PIC :

```bash
cd deploy-argocd

ansible-playbook -i inventory.ini playbook-crds.yml
ansible-playbook -i inventory.ini playbook-helm-release.yml
ansible-playbook -i inventory.ini playbook-route.yml
```

**Ordre strict à respecter** :

```
playbook-crds.yml
        ↓ (CRDs Application/ApplicationSet/AppProject établies
           avant que Helm n'installe les ressources qui en dépendent)
playbook-helm-release.yml
        ↓ (le Service argocd-server doit exister
           avant que le HTTPRoute qui le référence ne soit créé)
playbook-route.yml
```

> **Prérequis externe** : `platform-gateway` (namespace `gateway-system`) et le `Certificate argocd-tls` doivent déjà exister avant `playbook-route.yml` — tous deux créés par le composant Gateway API/NGF, pas par celui-ci.

> **Idempotence des CRDs** : `playbook-crds.yml` peut être rejoué sans risque à chaque upgrade de version du chart — c'est le mécanisme qui garantit la mise à jour des CRDs que `crds.install: false` empêche Helm de faire lui-même.

---

## 2️⃣ Vérifications

**Pods** :

```bash
kubectl get pods -n argocd
```

Attendu, tous `Running` : `argocd-application-controller-0` (StatefulSet, 1/1 — identité stable requise pour le sharding des clusters gérés), `argocd-applicationset-controller-*` (1/1), `argocd-dex-server-*` (1/1), `argocd-notifications-controller-*` (1/1), `argocd-redis-*` (2/2 — conteneurs `redis` + `redis-exporter`), `argocd-repo-server-*` (1/1), `argocd-server-*` (1/1).

**CRDs établies** :

```bash
kubectl get crd | grep argoproj.io
```

Attendu : `applications.argoproj.io`, `applicationsets.argoproj.io`, `appprojects.argoproj.io`, toutes présentes.

**Images réellement utilisées** (toutes doivent pointer vers le registre Mirror) :

```bash
kubectl get pods -n argocd -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}  {.name}: {.image}{"\n"}{end}{end}'
```

4 images au total : `argocd` (partagée par controller/server/repo-server/applicationset-controller/notifications-controller/dex-init), `dex`, `redis`, `redis_exporter`.

**ServiceMonitor scrapé par Prometheus** (voir piège ci-dessus — ne pas se fier uniquement à l'existence de la ressource) :

```bash
kubectl get servicemonitor -n argocd -o jsonpath='{range .items[*]}{.metadata.name}{"  ->  "}{.metadata.labels.release}{"\n"}{end}'
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &
curl -s 'http://localhost:9090/api/v1/query?query=up%7Bnamespace%3D%22argocd%22%7D' | python3 -m json.tool
kill %1
```

Attendu : `release: kube-prometheus-stack` sur les 4 `ServiceMonitor`, et 4 séries `up` à `1` (application-controller, applicationset-controller, repo-server, server).

**HTTPRoute** :

```bash
kubectl describe httproute argocd-route -n argocd
```

Attendu : `Accepted: True`, `ResolvedRefs: True` sur la condition de parent.

**Config serveur effective** (confirme que les deux correctifs anti-boucle sont bien appliqués) :

```bash
kubectl get deployment argocd-server -n argocd -o jsonpath='{.spec.template.spec.containers[0].args}'; echo
kubectl logs -n argocd deployment/argocd-server --tail=5 | grep "serving on port"
```

Attendu : `["--insecure"]` dans les args, et le log de démarrage affichant `url: https://<argocd_hostname>` (pas `argocd.example.com`).

---

## 3️⃣ Test fonctionnel

Test end-to-end sur le hostname ArgoCD, via `platform-gateway` :

```bash
curl -k --resolve argocd.dev.local:443:<IP_PLATFORM_GATEWAY> https://argocd.dev.local/ -I
```

Attendu : `HTTP/2 200` (un `307` en boucle signale que `global.domain` ou `server.extraArgs` manque — voir pièges ci-dessus).

Mot de passe admin initial :

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

User : `admin`. **À changer après le premier login** (UI : *User Info → Update Password*, ou `argocd account update-password`), puis supprimer le secret initial (`kubectl -n argocd delete secret argocd-initial-admin-secret`) — il n'est pas régénéré une fois le mot de passe défini, donc pas de risque d'accès de secours resté ouvert.

---

## Contenu des playbooks

**`playbook-crds.yml`**

- *Apply ArgoCD CustomResourceDefinitions* : applique les 3 CRDs (server-side apply, lues depuis `bundle/crds/` sur la VM PIC via `lookup('file', ...)` — rendu Helm déjà effectué en amont lors de la préparation du bundle, pas de template Jinja2 ici)
- *Wait for CRDs to be established* : boucle `until` sur la condition `Established`, 10 tentatives / 3s

**`playbook-helm-release.yml`**

- *Create argocd namespace* : crée le namespace `argocd`
- *Add internal Helm repository* : ajoute le dépôt Helm dédié pointant vers la VM Mirror
- *Render values* : génère `values-argocd.yaml` à partir du template — `crds.install: false` (CRDs gérées exclusivement par le playbook précédent), overrides des 4 images, `global.domain`, `server.extraArgs: ["--insecure"]`, mode non-HA (`redis.enabled: true` / `redis-ha.enabled: false`), `metrics.serviceMonitor.additionalLabels.release: kube-prometheus-stack` sur les 4 composants
- *Deploy via Helm* : installe/upgrade le release `argocd`
- *Verify pods Running* : liste et affiche la phase de chaque pod du namespace

**`playbook-route.yml`**

- *Render HTTPRoute manifest* : rendu délégué à `localhost`
- *Apply HTTPRoute* : crée le `HTTPRoute argocd-route` dans `argocd`, `parentRefs` vers `platform-gateway`/`gateway-system`, une seule règle `/` → backend `argocd-server:80`. Ne crée ni `Certificate` ni `Gateway`.

---

## Variables clés (`group_vars/all.yml`)

```yaml
# --- Chart / release ---
argocd_chart_version: "10.3.0"
argocd_app_version: "v3.5.0"
argocd_namespace: "argocd"
argocd_release_name: "argocd"

# --- Registre et dépôt Helm internes ---
image_registry: "192.168.1.20:5000/argocd"
argocd_helm_repo_name: "argocd-local"
argocd_helm_repo_url: "http://192.168.1.20:8080"

# --- Images (4 au total) ---
argocd_images:
  argocd:        { repository: "{{ image_registry }}/argocd",         tag: "v3.5.0" }
  dex:           { repository: "{{ image_registry }}/dex",            tag: "v2.45.1" }
  redis:         { repository: "{{ image_registry }}/redis",          tag: "8.6.4-alpine" }
  redis_exporter:{ repository: "{{ image_registry }}/redis_exporter", tag: "v1.88.0" }

# --- HA (voir section dédiée plus bas) ---
ha_enabled: false   # redis simple sur le cluster de test ; redis-ha pressenti pour la cible

# --- Exposition (via platform-gateway) ---
argocd_hostname: "argocd.dev.local"

# --- Monitoring ---
monitoring_enabled: true
prometheus_release_label: "kube-prometheus-stack"

# --- CRDs gérées séparément du Helm release ---
argocd_crds_enabled: false
argocd_crds_dir: "{{ playbook_dir }}/../bundle/crds"
argocd_crds_files:
  - crd-application.yaml
  - crd-applicationset.yaml
  - crd-appproject.yaml
```

---

## Pratique : extraction des CRDs (dans `templates/`, pas `crds/`)

Puisque `helm show crds` ne fonctionne pas pour ce chart (voir piège en introduction), les CRDs doivent être rendues individuellement avant d'être copiées dans `bundle/crds/` :

```bash
cd bundle
helm template argocd charts/argo-cd --dry-run=client --show-only templates/crds/crd-application.yaml      > crds/crd-application.yaml
helm template argocd charts/argo-cd --dry-run=client --show-only templates/crds/crd-applicationset.yaml   > crds/crd-applicationset.yaml
helm template argocd charts/argo-cd --dry-run=client --show-only templates/crds/crd-appproject.yaml       > crds/crd-appproject.yaml

# Vérifier l'absence de syntaxe Helm résiduelle avant de considérer les fichiers utilisables
grep -c "{{" crds/*.yaml   # doit afficher 0 partout
```

Deux des trois fichiers dépassent largement la limite d'annotation `kubectl apply` de 256 Ko (`crd-applicationset.yaml` ≈ 1,4 Mo, `crd-application.yaml` ≈ 400 Ko) — d'où le recours systématique à `server_side_apply` dans `playbook-crds.yml`, jamais à un `kubectl apply -f` classique.

---

## Pratique : dimensionnement HA pour la cible (cluster à workers multiples)

Le déploiement actuel tourne en mode **non-HA** (`redis.enabled: true`, `redis-ha.enabled: false`) — choix dicté par le cluster de test (2-3 workers, contrainte d'anti-affinité stricte du sous-chart `redis-ha` sur 3 nœuds distincts). Sur la cible (9 workers), rien n'empêche de réactiver le mode HA complet :

```yaml
ha_enabled: true
redis_ha_replicas: 3      # pas de réduction nécessaire, assez de workers pour l'anti-affinité stricte
haproxy_replicas: 3
controller_replicas: 2    # sharding des clusters gérés — intérêt limité si un seul cluster cible, mais résilience du pod
server_replicas: 2
repo_server_replicas: 2
applicationset_replicas: 2
```

Le template `values-argocd.yaml.j2` n'a besoin d'aucune réécriture pour ce basculement — uniquement les valeurs de `group_vars/all.yml` à ajuster, puis rejouer `playbook-helm-release.yml`. Le sous-chart `redis-ha` devra être réactivé dans le template (`redis.enabled: false` / `redis-ha.enabled: true`, avec les images/tags du sous-chart `redis-ha` qui diffèrent légèrement du mode simple — vérifier `charts/argo-cd/charts/redis-ha/values.yaml` avant de basculer, ne pas réutiliser telles quelles les clés `argocd_images.redis`/`redis_exporter` du mode non-HA).
