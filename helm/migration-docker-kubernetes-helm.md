# Migration Docker → Kubernetes avec Helm et Argo CD

**Application de Lilian : front, API, MongoDB** · mise à jour du 10/11/2026

---

## 0. Où on en est

| Sujet | Statut | Conséquence |
|---|---|---|
| L'API est-elle stateless ? | **Oui** | 2 réplicas possibles, mises à jour sans coupure |
| Outil de déploiement cible | **Argo CD, en GitOps** | Pas de `helm install` sur le cluster cible : la CI commite, Argo CD applique (§ 11) |
| Type de cluster cible | **Kubernetes classique** | k3d en local pour mettre au point le chart |
| Transactions ou change streams Mongo ? | À vérifier dans le code (§ 3.1) | Mongo standalone ou replica set |
| Base de données dans le cluster ? | À demander à l'admin (§ 3.2) | Ce que contient le chart |
| Mécanisme de secrets ? | À demander à l'admin Argo CD (§ 3.3) | Nom des Secrets à référencer |
| Ingress controller cible ? | À demander à l'admin (§ 9) | `ingressClassName`, annotations |

**Action immédiate : récupérer et lire le docker-compose de Lilian** (§ 1). Tout le reste en découle.

---

## 1. Inventaire du docker-compose (avec Lilian)

```bash
docker compose config               # le compose "résolu", variables substituées
docker compose ps                   # noms réels des conteneurs
docker compose exec mongo mongod --version
```

Pour chaque image, lire ce qu'elle attend : c'est ce qui alimente les templates.

```bash
docker inspect --format 'User={{.Config.User}} Ports={{.Config.ExposedPorts}} Env={{.Config.Env}}' api:1.0.0
```

Un `User` vide signifie que l'image tourne en root (voir § 8).

Grille à remplir pendant la séance :

| | front | api | mongo |
|---|---|---|---|
| Image et tag | | | |
| Port d'écoute | | | |
| Utilisateur (root ?) | | | |
| Variables non sensibles | | | |
| Variables sensibles | | | |
| Volumes | | | |
| Healthcheck | | | |
| Dépend de | | | |

---

## 2. Équivalences docker-compose → Kubernetes

| docker-compose | Kubernetes, dans le chart |
|---|---|
| service `front` / `api` | Deployment + Service |
| service `mongo` | StatefulSet + Service headless + PVC (ou un opérateur, ou Mongo hors cluster) |
| `environment`, `env_file` | ConfigMap pour le non sensible, Secret pour les mots de passe |
| `ports` exposés vers l'extérieur | Ingress |
| `volumes` nommés | PersistentVolumeClaim |
| `volumes` en bind mount d'un fichier de config | ConfigMap montée en volume |
| `healthcheck` | probes liveness / readiness |
| `depends_on` | **N'existe pas** : readiness probe + API qui réessaie la connexion à Mongo (+ sync wave Argo CD) |
| nom de service (`mongo:27017`) | Nom DNS du Service (`monapp-mongo:27017`), rendu paramétrable |
| `restart: always` | Natif : Kubernetes relance les pods |
| `networks` | Inutile (DNS des Services) ; NetworkPolicy si cloisonnement exigé |
| `build:` | Image construite en CI, poussée dans Nexus avec un tag immuable |

`kompose convert` peut générer un premier jet à partir du compose : utile comme checklist, à ne pas garder tel quel.

---

## 3. Décisions d'architecture

### 3.1 MongoDB : standalone ou replica set

Mongo peut tourner de deux façons :
- **standalone** : une seule instance, probablement le cas dans Docker aujourd'hui ;
- **replica set** : plusieurs instances qui se recopient les données, une principale et des secondaires qui prennent le relais en cas de panne.

Deux fonctions de Mongo n'existent qu'en replica set :
- **les transactions** : écrire plusieurs documents en « tout ou rien » (retirer un article du stock et créer la commande, ou ni l'un ni l'autre) ;
- **les change streams** : être prévenu en temps réel quand une donnée change.

Si l'API utilise l'une d'elles sur un Mongo standalone, elle plante. Pour vérifier :

```bash
grep -rnE "startTransaction|withTransaction|\.watch\(" src/
```

Indice rapide : si le compose ne lance pas Mongo avec `--replSet` et que l'appli fonctionne, elle ne s'en sert probablement pas (probablement seulement : le code peut exister sans avoir encore été exécuté). Si elle s'en sert, ce n'est pas bloquant : un replica set peut tourner avec un seul membre en dev, mais le StatefulSet du § 6 devra être adapté.

### 3.2 Base de données dans le cluster ou pas

Kubernetes tue, recrée et déplace les pods sans prévenir. C'est parfait pour l'API, délicat pour une base : les données doivent survivre à tout ça, être sauvegardées, restaurées et mises à jour sans perte. Beaucoup d'équipes Kubernetes refusent donc les bases dans le cluster, ou les encadrent.

| Option | Qui gère sauvegardes, versions, pannes | Pour | Contre |
|---|---|---|---|
| **StatefulSet dans le chart** (§ 6) | Toi | Simple, autonome | Ni haute dispo ni sauvegarde intégrées : bien pour le dev, pas pour la prod |
| **Opérateur MongoDB** | L'opérateur | Réplication et bascule automatiques ; on décrit « un Mongo à 3 membres » en YAML | Installation avec droits admin sur tout le cluster : à demander s'il en existe déjà un |
| **Mongo hors cluster** (VM, équipe DBA) | L'équipe qui l'héberge | Souvent le plus sain en on-premise | Dépendance à une autre équipe ; le chart ne gère que front et API (`mongo.enabled: false`) |

À propos des solutions toutes faites :
- **Chart Bitnami : à éviter.** Depuis le 28 août 2025, Bitnami ne publie plus de nouvelles images ni de nouveaux charts sur Docker Hub ; l'offre gratuite est réservée au développement et limitée au tag « latest », et les anciennes images ont été déplacées dans un dépôt legacy qui ne reçoit plus de mises à jour.
- **Opérateur officiel** : MongoDB Controllers for Kubernetes (MCK) remplace les anciens opérateurs Community et Enterprise et gère MongoDB Community. Il exige une StorageClass sur le cluster.

**Recommandation** : StatefulSet du chart pour avancer en local et en dev ; décision prod (opérateur existant ou Mongo hors cluster) à prendre avec l'admin en parallèle.

### 3.3 Secrets

En GitOps, tout ce qu'Argo CD déploie est dans Git : **aucun mot de passe dans les fichiers values**, même encodé en base64 (ce n'est pas du chiffrement). Le Secret est créé hors du chart par le mécanisme branché sur le cluster (External Secrets avec Vault, Sealed Secrets, SOPS…), et le chart se contente de le référencer :

```yaml
api:
  existingSecret: monapp-api-credentials     # créé par External Secrets / Vault
```

Le chart du § 6 gère les deux cas : il crée le Secret à partir des values en local (`values-local.yaml`), et le référence sans le créer dès qu'`existingSecret` est renseigné.

---

## 4. Prérequis sur le poste

1. Installer `k3d`, `kubectl` et `helm`.
2. Créer le cluster, avec le port 80 de l'ingress exposé sur le port 8080 du poste :

   ```bash
   k3d cluster create monapp -p "8080:80@loadbalancer"
   kubectl get nodes
   ```

3. Donner des tags versionnés aux images (jamais `latest`), puis les importer dans le cluster :

   ```bash
   docker images                                  # repérer noms et tags exacts
   docker tag api:latest api:1.0.0
   docker tag front:latest front:1.0.0
   k3d image import front:1.0.0 api:1.0.0 -c monapp
   ```

   Pour Mongo, l'image officielle `mongo:7.0` n'a pas besoin d'être importée (à aligner sur la version utilisée dans Docker, cf. § 1).

4. Pour le cluster cible, pousser les images dans Nexus :

   ```bash
   docker tag api:1.0.0 nexus.mondomaine:8082/monapp/api:1.0.0
   docker push nexus.mondomaine:8082/monapp/api:1.0.0
   ```

---

## 5. Structure du chart

```bash
helm create monapp     # puis supprimer hpa, serviceaccount, tests et les templates d'exemple
```

```
monapp/
  Chart.yaml
  values.yaml          # valeurs par défaut
  values-local.yaml    # k3d sur le poste (mots de passe factices autorisés)
  values-dev.yaml      # environnement dev géré par Argo CD (aucun secret)
  values-prod.yaml     # environnement prod géré par Argo CD (aucun secret)
  templates/
    _helpers.tpl
    mongo-secret.yaml  mongo-service.yaml  mongo-statefulset.yaml
    api-configmap.yaml api-secret.yaml     api-deployment.yaml     api-service.yaml
    front-deployment.yaml  front-service.yaml
    ingress.yaml
```

> Le fichier pour k3d s'appelle `values-local.yaml` et non `values-dev.yaml` : `values-dev.yaml` sera lu par Argo CD pour l'environnement de dev partagé, où aucun mot de passe ne doit figurer.

---

## 6. Le chart complet

> Validation effectuée : rendu des templates avec les values local et prod, et contrôle des manifestes générés contre les schémas Kubernetes 1.31 (kubeconform, 18 ressources valides). Pas encore déployé sur un cluster réel : à confirmer au premier `helm upgrade --install` sur k3d.

Les valeurs marquées « à confirmer » (port de l'API, chemin de santé, noms de variables) viennent de l'inventaire du § 1.

### `Chart.yaml`

```yaml
apiVersion: v2
name: monapp
description: Front + API + MongoDB (migration depuis docker-compose)
type: application
version: 0.1.0        # version du chart, à incrémenter à chaque modification des templates
appVersion: "1.0.0"   # version de l'application (indicative)
```

### `values.yaml`

```yaml
# Valeurs par défaut, communes à tous les environnements.
# Les fichiers values-<env>.yaml ne surchargent que ce qui change.

global:
  imagePullSecrets: []          # ex. [{ name: nexus-pull }] si le registry est privé

mongo:
  enabled: true                 # false = Mongo hors cluster (VM, équipe DBA)
  externalHost: ""              # utilisé seulement si enabled=false
  port: 27017
  database: monapp
  image:
    repository: mongo           # image officielle, à remplacer par la golden image Nexus
    tag: "7.0"                  # version figée, jamais "latest"
  auth:
    rootUser: root
    rootPassword: ""            # obligatoire si existingSecret est vide
  existingSecret: ""            # Secret fourni par External Secrets / Vault (clés : MONGO_INITDB_ROOT_USERNAME, MONGO_INITDB_ROOT_PASSWORD)
  storage:
    size: 5Gi
    storageClass: ""            # vide = StorageClass par défaut du cluster
  resources:
    requests: { cpu: 250m, memory: 512Mi }
    limits:   { memory: 1Gi }

api:
  enabled: true
  replicas: 2                   # l'API est stateless : 2 réplicas possibles
  image:
    repository: monregistry/api
    tag: "1.0.0"
    pullPolicy: IfNotPresent
  port: 3000                    # port d'écoute réel du conteneur (docker inspect)
  readinessPath: /health        # "prête" : peut vérifier la connexion à Mongo
  livenessPath: /health         # "vivante" : ne doit pas dépendre de Mongo (endpoint distinct si /health teste Mongo)
  env: {}                       # variables non sensibles supplémentaires, ex. LOG_LEVEL: info
  secretEnv: {}                 # variables sensibles (dev local uniquement)
  existingSecret: ""            # en GitOps : nom du Secret créé hors chart
  resources:
    requests: { cpu: 100m, memory: 256Mi }
    limits:   { memory: 512Mi }

front:
  enabled: true
  replicas: 2
  image:
    repository: monregistry/front
    tag: "1.0.0"
    pullPolicy: IfNotPresent
  port: 8080                    # 8080 si nginx-unprivileged, 80 si nginx root
  healthPath: /
  resources:
    requests: { cpu: 50m, memory: 64Mi }
    limits:   { memory: 128Mi }

ingress:
  enabled: true
  className: ""                 # traefik sur k3d, nginx (ou autre) sur le cluster cible
  host: monapp.local
  annotations: {}               # réécriture de chemin, taille max d'upload, etc.
  tls: []                       # ex. [{ secretName: monapp-tls, hosts: [monapp.example] }]

# Sécurité appliquée à l'API et au front (compatible Pod Security "restricted")
podSecurityContext:
  runAsNonRoot: true
  seccompProfile: { type: RuntimeDefault }
containerSecurityContext:
  allowPrivilegeEscalation: false
  capabilities: { drop: [ALL] }
```

### `values-local.yaml` (k3d)

```yaml
# Surcharges pour le cluster k3d sur le poste. Mots de passe factices autorisés ici, nulle part ailleurs.
mongo:
  auth:
    rootPassword: devpassword
  storage:
    size: 1Gi

api:
  replicas: 1
  image:
    repository: api            # image importée avec "k3d image import"
    tag: "1.0.0"
  env:
    LOG_LEVEL: debug
  secretEnv:
    MONGO_USER: root
    MONGO_PASSWORD: devpassword

front:
  replicas: 1
  image:
    repository: front
    tag: "1.0.0"

ingress:
  className: traefik
  host: monapp.localhost
```

### `values-prod.yaml` (Argo CD ; `values-dev.yaml` suit le même modèle)

```yaml
# Surcharges pour un environnement géré par Argo CD. Aucun secret ici : tout est dans Git.
mongo:
  existingSecret: monapp-mongo-credentials     # créé par External Secrets / Vault
  storage:
    size: 20Gi
    storageClass: ""                           # à demander à l'admin

api:
  image:
    repository: nexus.mondomaine:8082/monapp/api
    tag: "1.0.0"                               # mis à jour par la CI (commit)
  existingSecret: monapp-api-credentials

front:
  image:
    repository: nexus.mondomaine:8082/monapp/front
    tag: "1.0.0"

global:
  imagePullSecrets:
    - name: nexus-pull

ingress:
  className: nginx                             # à confirmer avec l'admin
  host: monapp.mondomaine
  tls:
    - secretName: monapp-tls
      hosts: [monapp.mondomaine]
```

### `templates/_helpers.tpl`

```yaml
{{/* Nom de base des ressources : <release> (ex. monapp-dev) */}}
{{- define "monapp.fullname" -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/* Labels communs */}}
{{- define "monapp.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{- end -}}

{{/* Labels de sélection d'un composant : include "monapp.selectorLabels" (dict "ctx" . "component" "api") */}}
{{- define "monapp.selectorLabels" -}}
app.kubernetes.io/name: {{ .ctx.Chart.Name }}
app.kubernetes.io/instance: {{ .ctx.Release.Name }}
app.kubernetes.io/component: {{ .component }}
{{- end -}}

{{/* Adresse de Mongo : Service du chart, ou hôte externe */}}
{{- define "monapp.mongo.host" -}}
{{- if .Values.mongo.enabled -}}
{{ include "monapp.fullname" . }}-mongo
{{- else -}}
{{ required "mongo.externalHost est obligatoire quand mongo.enabled=false" .Values.mongo.externalHost }}
{{- end -}}
{{- end -}}

{{/* Nom du Secret de l'API */}}
{{- define "monapp.api.secretName" -}}
{{- .Values.api.existingSecret | default (printf "%s-api" (include "monapp.fullname" .)) -}}
{{- end -}}

{{/* Nom du Secret de Mongo */}}
{{- define "monapp.mongo.secretName" -}}
{{- .Values.mongo.existingSecret | default (printf "%s-mongo" (include "monapp.fullname" .)) -}}
{{- end -}}
```

### `templates/mongo-secret.yaml`

```yaml
{{- if and .Values.mongo.enabled (not .Values.mongo.existingSecret) }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "monapp.mongo.secretName" . }}
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
type: Opaque
stringData:
  MONGO_INITDB_ROOT_USERNAME: {{ .Values.mongo.auth.rootUser | quote }}
  MONGO_INITDB_ROOT_PASSWORD: {{ required "mongo.auth.rootPassword est obligatoire (ou mongo.existingSecret)" .Values.mongo.auth.rootPassword | quote }}
{{- end }}
```

### `templates/mongo-service.yaml`

```yaml
{{- if .Values.mongo.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "monapp.fullname" . }}-mongo
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: mongo
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  clusterIP: None                 # Service "headless", requis par le StatefulSet
  selector:
    {{- include "monapp.selectorLabels" (dict "ctx" . "component" "mongo") | nindent 4 }}
  ports:
    - name: mongo
      port: {{ .Values.mongo.port }}
      targetPort: mongo
{{- end }}
```

### `templates/mongo-statefulset.yaml`

```yaml
{{- if .Values.mongo.enabled }}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "monapp.fullname" . }}-mongo
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: mongo
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  serviceName: {{ include "monapp.fullname" . }}-mongo
  replicas: 1                     # standalone : ne pas augmenter sans passer en replica set
  selector:
    matchLabels:
      {{- include "monapp.selectorLabels" (dict "ctx" . "component" "mongo") | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "monapp.selectorLabels" (dict "ctx" . "component" "mongo") | nindent 8 }}
    spec:
      {{- with .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 999              # utilisateur "mongodb" de l'image officielle
        runAsGroup: 999
        fsGroup: 999                # rend le volume inscriptible par mongodb
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: mongo
          image: "{{ .Values.mongo.image.repository }}:{{ .Values.mongo.image.tag }}"
          ports:
            - name: mongo
              containerPort: 27017
          envFrom:
            - secretRef:
                name: {{ include "monapp.mongo.secretName" . }}
          readinessProbe:
            exec:
              command: ["mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
          livenessProbe:
            tcpSocket:
              port: mongo
            initialDelaySeconds: 30
            periodSeconds: 20
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
          resources:
            {{- toYaml .Values.mongo.resources | nindent 12 }}
          volumeMounts:
            - name: data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        {{- with .Values.mongo.storage.storageClass }}
        storageClassName: {{ . }}
        {{- end }}
        resources:
          requests:
            storage: {{ .Values.mongo.storage.size }}
{{- end }}
```

### `templates/api-configmap.yaml`

```yaml
{{- if .Values.api.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "monapp.fullname" . }}-api
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: api
data:
  # Noms à aligner sur les variables réellement lues par l'API (cf. docker inspect)
  MONGO_HOST: {{ include "monapp.mongo.host" . | quote }}
  MONGO_PORT: {{ .Values.mongo.port | quote }}
  MONGO_DB: {{ .Values.mongo.database | quote }}
  {{- range $key, $value := .Values.api.env }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
```

### `templates/api-secret.yaml`

```yaml
{{- if and .Values.api.enabled (not .Values.api.existingSecret) }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "monapp.api.secretName" . }}
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: api
type: Opaque
{{- with .Values.api.secretEnv }}
stringData:
  {{- range $key, $value := . }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
{{- end }}
```

### `templates/api-deployment.yaml`

```yaml
{{- if .Values.api.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "monapp.fullname" . }}-api
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: api
spec:
  replicas: {{ .Values.api.replicas }}
  selector:
    matchLabels:
      {{- include "monapp.selectorLabels" (dict "ctx" . "component" "api") | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "monapp.selectorLabels" (dict "ctx" . "component" "api") | nindent 8 }}
      annotations:
        # envFrom ne recharge pas une ConfigMap modifiée : ce hash force le redémarrage des pods
        checksum/config: {{ include (print $.Template.BasePath "/api-configmap.yaml") . | sha256sum }}
    spec:
      {{- with .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: api
          image: "{{ .Values.api.image.repository }}:{{ .Values.api.image.tag }}"
          imagePullPolicy: {{ .Values.api.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.api.port }}
          envFrom:
            - configMapRef:
                name: {{ include "monapp.fullname" . }}-api
            - secretRef:
                name: {{ include "monapp.api.secretName" . }}
          # readiness : "prête à recevoir du trafic" (peut vérifier Mongo)
          readinessProbe:
            httpGet:
              path: {{ .Values.api.readinessPath }}
              port: http
            periodSeconds: 10
            failureThreshold: 3
          # liveness : "le processus répond" (ne doit PAS dépendre de Mongo)
          livenessProbe:
            httpGet:
              path: {{ .Values.api.livenessPath }}
              port: http
            initialDelaySeconds: 20
            periodSeconds: 20
            failureThreshold: 3
          securityContext:
            {{- toYaml .Values.containerSecurityContext | nindent 12 }}
          resources:
            {{- toYaml .Values.api.resources | nindent 12 }}
{{- end }}
```

### `templates/api-service.yaml`

```yaml
{{- if .Values.api.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "monapp.fullname" . }}-api
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: api
spec:
  selector:
    {{- include "monapp.selectorLabels" (dict "ctx" . "component" "api") | nindent 4 }}
  ports:
    - name: http
      port: 80
      targetPort: http
{{- end }}
```

### `templates/front-deployment.yaml`

```yaml
{{- if .Values.front.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "monapp.fullname" . }}-front
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: front
spec:
  replicas: {{ .Values.front.replicas }}
  selector:
    matchLabels:
      {{- include "monapp.selectorLabels" (dict "ctx" . "component" "front") | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "monapp.selectorLabels" (dict "ctx" . "component" "front") | nindent 8 }}
    spec:
      {{- with .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: front
          image: "{{ .Values.front.image.repository }}:{{ .Values.front.image.tag }}"
          imagePullPolicy: {{ .Values.front.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.front.port }}
          readinessProbe:
            httpGet:
              path: {{ .Values.front.healthPath }}
              port: http
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: {{ .Values.front.healthPath }}
              port: http
            initialDelaySeconds: 10
            periodSeconds: 20
          securityContext:
            {{- toYaml .Values.containerSecurityContext | nindent 12 }}
          resources:
            {{- toYaml .Values.front.resources | nindent 12 }}
{{- end }}
```

### `templates/front-service.yaml`

```yaml
{{- if .Values.front.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "monapp.fullname" . }}-front
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: front
spec:
  selector:
    {{- include "monapp.selectorLabels" (dict "ctx" . "component" "front") | nindent 4 }}
  ports:
    - name: http
      port: 80
      targetPort: http
{{- end }}
```

### `templates/ingress.yaml`

```yaml
{{- if and .Values.ingress.enabled (or .Values.api.enabled .Values.front.enabled) }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "monapp.fullname" . }}
  labels:
    {{- include "monapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- with .Values.ingress.className }}
  ingressClassName: {{ . }}
  {{- end }}
  {{- with .Values.ingress.tls }}
  tls:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host | quote }}
      http:
        paths:
          {{- if .Values.api.enabled }}
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: {{ include "monapp.fullname" . }}-api
                port:
                  name: http
          {{- end }}
          {{- if .Values.front.enabled }}
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "monapp.fullname" . }}-front
                port:
                  name: http
          {{- end }}
{{- end }}
```

---

## 7. Monter composant par composant (en local)

Un interrupteur `enabled` par composant permet d'avancer avec un seul chart. On ne passe au composant suivant que quand le précédent est `Running` et `Ready`.

**Boucle à chaque étape :**

```bash
helm lint ./monapp -f monapp/values-local.yaml
helm template monapp ./monapp -f monapp/values-local.yaml    # relire le YAML généré
helm upgrade --install monapp ./monapp -n monapp --create-namespace -f monapp/values-local.yaml <options de l'étape>
kubectl get pods -n monapp -w
```

**Étape A : Mongo seul**

```bash
helm upgrade --install monapp ./monapp -n monapp --create-namespace \
  -f monapp/values-local.yaml --set api.enabled=false --set front.enabled=false
kubectl exec -it -n monapp monapp-mongo-0 -- mongosh -u root -p devpassword --eval 'db.adminCommand("ping")'
```

**Étape B : Mongo + API**

```bash
helm upgrade --install monapp ./monapp -n monapp -f monapp/values-local.yaml --set front.enabled=false
kubectl port-forward -n monapp svc/monapp-api 8081:80
curl -i http://localhost:8081/health        # dans un autre terminal
```

**Étape C : tout**

```bash
helm upgrade --install monapp ./monapp -n monapp -f monapp/values-local.yaml
# puis ouvrir http://monapp.localhost:8080
```

**Diagnostic :**

| Symptôme | Commande | Cause fréquente |
|---|---|---|
| `ImagePullBackOff` | `kubectl describe pod <pod> -n monapp` | Image non importée dans k3d, mauvais tag, `imagePullSecret` manquant |
| `CrashLoopBackOff` | `kubectl logs <pod> -n monapp --previous` | Variable d'environnement manquante, Mongo injoignable sans retry |
| `CreateContainerConfigError` | `kubectl describe pod <pod> -n monapp` | Secret ou ConfigMap absent, ou image root avec `runAsNonRoot` (§ 8) |
| `Pending` | `kubectl get pvc -n monapp` | Pas de StorageClass, ressources insuffisantes |
| `Running` mais pas `Ready` | `kubectl describe pod <pod> -n monapp` | Readiness probe en échec : mauvais port ou mauvais chemin |
| Erreur 404 / 502 via l'Ingress | `kubectl get ingress,endpoints -n monapp` | Service sans endpoint, `ingressClassName` erroné, chemin `/api` non réécrit |

Dans 9 cas sur 10, Helm n'y est pour rien : c'est l'image ou sa configuration.

**Tests fonctionnels** : un pod `Ready` prouve seulement que le processus répond. À chaque étape, lancer aussi le test fourni par Lilian pour le composant (§ 13, questions 15 à 18). Plus tard, la requête de bout en bout de l'API pourra être automatisée dans un Job lancé par Argo CD après chaque synchronisation (hook `PostSync`), qui échoue si le résultat n'est pas le bon.

---

## 8. Probes, ressources, sécurité

- **Readiness probe** : « le pod peut recevoir du trafic ». Elle peut vérifier la connexion à Mongo. C'est elle qui remplace `depends_on` : tant que l'API n'est pas prête, l'Ingress ne lui envoie rien.
- **Liveness probe** : « le processus répond ». Elle ne doit **pas** dépendre de Mongo, sinon une panne de Mongo fait redémarrer en boucle tous les pods de l'API. Si `/health` teste Mongo, demander à Lilian un endpoint distinct (`livenessPath`).
- **Requests / limits** : les requests réservent la capacité, la limite mémoire évite qu'un pod emporte le nœud. Les valeurs du `values.yaml` sont des points de départ, à ajuster avec la consommation observée.
- **Sécurité** : le chart applique `runAsNonRoot`, `allowPrivilegeEscalation: false` et le retrait des capabilities, ce qu'exige le Pod Security « restricted ». Si une image tourne en root, le pod est refusé (`CreateContainerConfigError`). Contournement **temporaire, en local uniquement** :

  ```yaml
  # values-local.yaml
  podSecurityContext:
    runAsNonRoot: false
  ```

  La vraie correction est dans l'image : ajouter un `USER` dans le Dockerfile, et pour un front nginx passer à `nginxinc/nginx-unprivileged`, qui écoute sur le port 8080.

---

## 9. Ingress

Une seule règle : `/` vers le front, `/api` vers l'API. `ingressClassName` est dans les values : `traefik` sur k3d, autre chose sur le cluster cible.

**Trois points à vérifier :**

1. **URL de l'API dans le front.** Un front React, Vue ou Angular ne tourne pas dans le pod : nginx sert des fichiers JavaScript, et c'est le **navigateur de l'utilisateur** qui exécute le code et appelle l'API. Deux cas :
   - **URL injectée au build** (`VITE_API_URL`, `REACT_APP_API_URL`…) : elle est écrite en dur dans le JavaScript de l'image. Une ConfigMap sur le pod ne change rien, et il faut une image par environnement, ce qui casse le principe « même image, même checksum, du dev à la prod ». Piège classique : image construite avec `http://localhost:3000`, et en prod le navigateur cherche l'API sur le poste de l'utilisateur.
   - **URL lue au runtime** : un script de démarrage génère un `config.js` à partir des variables d'environnement, ou l'appli charge un `/config.json` au lancement. Même image partout, seule la ConfigMap change par environnement.

   Dans tous les cas, la meilleure solution est l'**URL relative `/api`** : le navigateur appelle le même domaine que le front, l'Ingress route, il n'y a rien à configurer et pas de CORS. L'adresse interne `http://monapp-api` ne peut pas fonctionner : elle n'existe que dans le cluster, pas dans le navigateur.

   Vérification sans attendre Lilian, en cherchant les URL écrites dans l'image :

   ```bash
   docker run --rm --entrypoint sh front:1.0.0 -c \
     'grep -rhoE "https?://[a-zA-Z0-9.:/_-]+" /usr/share/nginx/html | sort -u'
   ```

   Si `http://localhost:3000` ou un nom de serveur apparaît, l'URL est figée au build.
2. **Préfixe `/api`.** L'Ingress transmet le chemin tel quel : l'API reçoit `/api/users`. Si elle attend `/users`, il faut une réécriture, propre à chaque contrôleur. Le plus simple et le plus portable est de configurer l'API pour servir sous `/api`. À défaut, sur Traefik :

   ```yaml
   # middleware.yaml, à appliquer avec kubectl apply -f (ou à ajouter aux templates)
   apiVersion: traefik.io/v1alpha1
   kind: Middleware
   metadata:
     name: strip-api
     namespace: monapp
   spec:
     stripPrefix:
       prefixes: [/api]
   ```

   ```yaml
   # values-local.yaml : format <namespace>-<nom>@kubernetescrd
   ingress:
     annotations:
       traefik.ingress.kubernetes.io/router.middlewares: monapp-strip-api@kubernetescrd
   ```

   Le front n'est pas affecté : ses chemins ne commencent pas par `/api`.

3. **Contrôleur du cluster cible.** Le projet ingress-nginx de la communauté Kubernetes a été arrêté en mars 2026 : plus aucune version, ni correctif ni patch de sécurité. Si le cluster cible l'utilise encore, le signaler à l'admin ; les remplaçants recommandés sont Gateway API ou un autre contrôleur Ingress maintenu.
4. **Pages profondes du front.** Avec une SPA, un rafraîchissement sur `/commandes/42` renvoie 404 si nginx ne renvoie pas `index.html` pour les routes inconnues. La configuration nginx de l'image doit contenir `try_files $uri $uri/ /index.html;`.

---

## 10. Migration des données

On l'oublie souvent. Vérifier d'abord que la version de Mongo est la même des deux côtés (`mongod --version`, § 1).

```bash
# 1. Dump depuis Docker (-T : pas de terminal, indispensable pour rediriger la sortie)
docker compose exec -T mongo sh -c \
  'mongodump --archive --gzip -u root -p "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin' > dump.gz

# 2. Restauration dans le pod
kubectl exec -i -n monapp monapp-mongo-0 -- sh -c \
  'mongorestore --archive --gzip -u root -p "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin' < dump.gz

# 3. Contrôle : nombre de documents par collection, à comparer des deux côtés
kubectl exec -n monapp monapp-mongo-0 -- sh -c \
  'mongosh --quiet -u root -p "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin monapp \
   --eval "db.getCollectionNames().forEach(c => print(c, db[c].countDocuments()))"'
```

Si le compose ne définit pas `MONGO_INITDB_ROOT_PASSWORD` dans le conteneur Mongo, remplacer la variable par le mot de passe réel.

**Déroulé :**
1. Répétition en dev pour valider la procédure et mesurer sa durée.
2. Le jour de la bascule : arrêter l'API côté Docker (plus aucune écriture), dump, restauration, contrôle des volumes, bascule du DNS vers l'Ingress.
3. Garder l'environnement Docker arrêté mais intact quelques jours, comme retour arrière.

---

## 11. Passage en GitOps avec Argo CD

> **Correction par rapport à la première version de cette note** : avec Argo CD, la CI ne lance **pas** `helm upgrade --install` sur le cluster cible. Argo CD rend lui-même le chart et l'applique ; la CI se contente de mettre à jour Git.

### Dépôt de déploiement

```
monapp-deploy/
  charts/monapp/          # le chart du § 6, avec values-dev.yaml et values-prod.yaml
  argocd/
    application-dev.yaml
    application-prod.yaml
```

Pour démarrer, le chart vit dans Git, à côté de ses values. Plus tard, il pourra être packagé (`helm package`) et publié dans Nexus avec une version immuable, Argo CD pointant alors sur le dépôt Helm Nexus.

### Application Argo CD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monapp-dev
  namespace: argocd
spec:
  project: monapp                      # AppProject fourni par l'admin Argo CD
  source:
    repoURL: https://gitlab.mondomaine/monapp/monapp-deploy.git
    targetRevision: main
    path: charts/monapp
    helm:
      releaseName: monapp              # sans ça, les ressources prennent le nom de l'Application (monapp-dev-*)
      valueFiles:
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: monapp-dev
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    # À activer une fois le déploiement stabilisé :
    # automated:
    #   prune: true
    #   selfHeal: true
```

### Politique de synchronisation (`syncPolicy`)

Elle décide **quand** Argo CD applique ce qui est dans Git.

| Réglage | Comportement | Usage conseillé |
|---|---|---|
| Manuel | Après un commit, l'application passe en « OutOfSync » ; quelqu'un clique sur Sync ou lance `argocd app sync` | Mise au point ; prod si l'organisation l'exige |
| Automatique | Argo CD applique dès qu'il voit le commit (interrogation de Git toutes les 3 minutes environ, immédiate avec un webhook) | Dev, puis prod avec fenêtres de synchronisation |
| + `prune` | Une ressource retirée de Git est supprimée du cluster | Utile, à activer en connaissance de cause |
| + `selfHeal` | Une modification manuelle dans le cluster est annulée : un `kubectl edit` de débogage est écrasé en quelques secondes | Prod ; à suspendre le temps d'un débogage en dev |
| Fenêtres de synchronisation | Définies dans l'AppProject par l'admin (ex. prod synchronisable le mardi de 20 h à 22 h) | Prod |

Commencer en manuel pour relire le diff avant chaque application. Conséquence à garder en tête : si la synchronisation automatique est interdite en prod, **merger dans Git ne veut pas dire déployé**. Il faut savoir qui a le droit de synchroniser, et quand : c'est ce qui organise la bascule du § 10.

### Chaîne de livraison

1. La CI construit l'image et la pousse dans Nexus avec un tag immuable.
2. Elle met à jour le tag dans le dépôt de déploiement et commite :

   ```bash
   yq -i '.api.image.tag = "1.0.1"' charts/monapp/values-dev.yaml
   git commit -am "api 1.0.1 en dev" && git push
   ```

3. Argo CD détecte le commit et synchronise (ou attend un clic, en mode manuel).
4. Promotion en prod : même tag, recopié dans `values-prod.yaml`, par merge request.

### Commit par la CI ou Argo CD Image Updater

C'est la question de savoir comment le nouveau tag arrive dans Git.

- **Commit par la CI** (chaîne ci-dessus) : simple, explicite, chaque déploiement est un commit traçable. Il faut un token CI avec droit d'écriture sur le dépôt de déploiement.
- **Argo CD Image Updater** : composant installé dans le cluster par l'admin, qui surveille Nexus et met à jour l'application quand un nouveau tag respecte une règle (par exemple « tout 1.x »).
  - En écriture dans Git, il fait le commit lui-même : on reste en GitOps.
  - En écriture directe dans l'Application, la version déployée n'est plus dans Git : **à éviter**.
  - Il faut que l'admin l'installe et le maintienne, que le cluster puisse lire Nexus, et que l'outil ait des droits d'écriture sur Git.

**Recommandation** : commit par la CI pour commencer, sans rien installer. Image Updater éventuellement plus tard, pour déployer automatiquement chaque build en dev. Dans tous les cas, promotion en prod par merge request.

### Ce qui change par rapport à `helm install`

- `helm list` n'affiche rien et **`helm rollback` ne sert plus** : on revient en arrière avec un `git revert`, ou par l'historique d'Argo CD.
- Éviter dans les templates la fonction `lookup` (ne fonctionne pas avec Argo CD) et les valeurs aléatoires comme `randAlphaNum` (régénérées à chaque synchronisation).
- **Ordre de démarrage** : les ressources Mongo portent l'annotation `argocd.argoproj.io/sync-wave: "-1"`. Argo CD les applique et attend qu'elles soient saines avant l'API et le front. Cela ne remplace ni la readiness probe ni le retry de l'API.
- **Noms des ressources** : sans `helm.releaseName`, Argo CD utilise le nom de l'Application comme nom de release (`monapp-dev-api` au lieu de `monapp-api`).
- **Données** : supprimer l'Application ne supprime pas les PVC créés par le StatefulSet. C'est voulu, mais à savoir.

---

## 12. Pièges en passant de compose à Kubernetes

- **Images root** refusées sous Pod Security « restricted » ; nginx root sur le port 80 → `nginx-unprivileged` sur 8080.
- **Bind mounts** (`./data:/data`) : pas d'équivalent direct, passer par un PVC (données) ou une ConfigMap (fichiers de config).
- **Ports publiés sur l'hôte** : ils disparaissent, tout passe par les Services et l'Ingress.
- **Tag `latest`** : impossible de savoir ce qui tourne, impossible de revenir en arrière proprement.
- **`MONGO_INITDB_ROOT_*`** n'est lu qu'à la première initialisation, volume vide. Changer le Secret ensuite ne change pas le mot de passe dans la base.
- **Taille du volume Mongo** : `volumeClaimTemplates` est immuable dans un StatefulSet. Prévoir la bonne taille dès le départ ; un agrandissement ultérieur se fait sur le PVC lui-même, si la StorageClass l'autorise.
- **ConfigMap modifiée** : `envFrom` ne recharge rien à chaud ; l'annotation `checksum/config` du Deployment de l'API force le redémarrage des pods.
- **Migrations de schéma au démarrage** : avec deux réplicas, elles tournent en concurrence. À sortir dans un Job, ou à protéger par un verrou.
- **Sync automatique avec `prune`** : une ressource retirée de Git est supprimée du cluster. À activer en connaissance de cause.

---

## 13. Questions ouvertes

### À Lilian (développeur)

| # | Question | Pourquoi | Réponse |
|---|---|---|---|
| 1 | L'API est-elle stateless (pas de session en mémoire, pas de fichier local, pas de tâche planifiée interne) ? | Plusieurs réplicas | **Oui** |
| 2 | L'API utilise-t-elle les transactions ou les change streams de Mongo ? | Standalone ou replica set (§ 3.1) | |
| 3 | Liste exhaustive des variables d'environnement, sensibles ou non, avec leurs valeurs par défaut | ConfigMap et Secret | |
| 4 | L'URL de l'API est-elle injectée dans le front au build ou au runtime ? | Ingress `/api` (§ 9) | |
| 5 | L'API sert-elle sous `/api` ou sous `/` ? | Réécriture de chemin (§ 9) | |
| 6 | Quels endpoints de santé ? Distinguent-ils « vivant » et « prêt » ? Vérifient-ils la connexion à Mongo ? | Probes (§ 8) | |
| 7 | Si Mongo n'est pas prêt au démarrage, l'API réessaie-t-elle ou plante-t-elle ? | Remplace `depends_on` | |
| 8 | Les images tournent-elles en non-root ? Écrivent-elles sur le disque, et où ? | Pod Security | |
| 9 | Y a-t-il des migrations de schéma au démarrage ? | Concurrence entre réplicas | |
| 10 | Des fichiers sont-ils stockés hors de Mongo (uploads) ? Le S3 est-il toujours d'actualité ? | PVC ou stockage objet | |
| 11 | Version de Mongo, volume de données, sauvegarde actuelle | Migration (§ 10) | |
| 12 | L'API s'arrête-t-elle proprement sur SIGTERM ? Les logs vont-ils sur stdout ? | Mises à jour sans perte, collecte des logs | |
| 13 | Consommation CPU et mémoire observée | Requests et limits | |
| 14 | Appels sortants (SMTP, API externes, SSO) ? | Proxy, NetworkPolicy, URL de callback | |

**Recommandations, question par question**

- **Q1 – Stateless.** Le « oui » est souvent donné trop vite : faire confirmer les trois points un par un (sessions, fichiers sur disque, tâches planifiées). Une tâche planifiée interne s'exécuterait une fois par réplica : la sortir dans un CronJob Kubernetes ou la protéger par un verrou. En prod, ajouter un PodDisruptionBudget `minAvailable: 1`.
- **Q2 – Transactions.** Si non : standalone acceptable en dev. Si oui : replica set dès le dev (un seul membre, `--replSet rs0` puis `rs.initiate()`, et `replicaSet=rs0` dans l'URI de connexion). Dans les deux cas, viser un replica set à 3 membres en prod, pour la haute disponibilité et des sauvegardes cohérentes : c'est l'argument pour un opérateur ou l'équipe DBA (§ 3.2).
- **Q3 – Variables.** Demander un `.env.example` versionné dans le dépôt de l'appli, avec pour chaque variable : obligatoire ou non, valeur par défaut, sensible ou non. Préférer des variables séparées (hôte, port, utilisateur, mot de passe) à une URI complète : une URI qui contient le mot de passe doit aller entièrement dans le Secret.
- **Q4 – URL de l'API.** Viser l'URL relative `/api`, à défaut une configuration au runtime ; refuser une image par environnement (§ 9). La commande `grep` du § 9 permet de vérifier avant la séance.
- **Q5 – Préfixe `/api`.** Demander que l'API serve sous `/api` (chemin de base configurable) plutôt que de réécrire le chemin dans l'Ingress : la réécriture dépend du contrôleur et serait à refaire à chaque changement de contrôleur.
- **Q6 – Santé.** Demander deux endpoints : `/health/live` (le processus répond, aucune dépendance) et `/health/ready` (Mongo joignable). Réponse en moins d'une seconde, sans authentification, sans ligne de log à chaque appel. En attendant, liveness en `tcpSocket` sur le port de l'API.
- **Q7 – Retry Mongo.** L'API doit réessayer avec un délai croissant au lieu de planter. Sinon, le pod passe quelques fois en `CrashLoopBackOff` avant de démarrer : tolérable, mais bruyant. Béquille possible en attendant le correctif : un initContainer qui attend que Mongo réponde.
- **Q8 – Non-root.** Dans le Dockerfile, `USER` avec un **UID numérique** (`USER 10001`, pas `USER appuser`) : avec un nom, Kubernetes ne peut pas vérifier `runAsNonRoot` et refuse le pod. Écritures limitées à `/tmp` ou à un répertoire dédié, monté en `emptyDir`.
- **Q9 – Migrations.** Les sortir dans un Job exécuté avant le déploiement (hook Argo CD `PreSync`), jamais en parallèle dans chaque réplica.
- **Q10 – Fichiers.** Si l'API écrit des fichiers sur disque, elle n'est **pas** stateless : contradiction avec Q1, à lever. Solutions : stockage objet compatible S3 (MinIO interne), GridFS dans Mongo, ou volume partagé RWX si la StorageClass le permet.
- **Q11 – Données.** Même version majeure de Mongo des deux côtés. Mesurer la durée du dump et de la restauration en répétition : c'est la durée minimale de l'interruption. Faire fixer par le métier la perte de données acceptable (RPO) et la durée d'arrêt acceptable (RTO), qui dimensionnent la sauvegarde en prod.
- **Q12 – Arrêt et logs.** Arrêt propre en moins de 30 secondes (délai par défaut avant l'arrêt forcé). Dans le Dockerfile, `CMD` en forme tableau (`CMD ["node", "server.js"]`) : en forme shell, le signal d'arrêt n'atteint souvent pas l'application. Logs sur stdout, en JSON si possible, jamais dans un fichier.
- **Q13 – Ressources.** Mesurer avec `docker stats` sous charge réelle. Requests proches de la consommation moyenne, limite mémoire au pic plus 30 %. Pas de limite CPU (source de ralentissements par bridage), sauf si l'admin l'impose. Pour Node ou Java, aligner la taille du tas sur la limite mémoire.
- **Q14 – Appels sortants.** Lister chaque destination (hôte, port) pour l'admin : proxy, NetworkPolicy. Pour un SSO, l'URL de callback contient le nom d'hôte de l'Ingress : à enregistrer auprès du fournisseur d'identité pour chaque environnement.

**Tests de bon fonctionnement, par composant.** Avant la séance, lire le bloc `healthcheck:` du compose : une partie des réponses y est peut-être déjà.

| # | Composant | Question | Pourquoi | Réponse |
|---|---|---|---|---|
| 15 | Front | Quelle URL ouvrir, et qu'est-ce qui doit s'afficher ? Le front appelle-t-il l'API dès le chargement ? | Valider l'Ingress et le chemin `/api` (§ 7, § 9) | |
| 16 | API | Une requête qui lit vraiment dans Mongo, avec le résultat attendu (ex. `GET /api/xxx` renvoie une liste non vide) | Test de bout en bout API → Mongo (§ 7) | |
| 17 | API | Faut-il une authentification ou un token pour l'appeler ? | Lancer ce test depuis un script ou la CI | |
| 18 | Mongo | Quelle base et quelles collections doivent exister ? Quelle collection compter pour vérifier que les données sont là ? | Contrôle après restauration (§ 10) | |

Le simple ping de Mongo (`db.adminCommand('ping')`) est standard et ne dépend pas de l'appli : il est déjà dans la readiness probe du chart.

- **Q15 – Front.** Test scriptable : `curl -fsS https://<hôte>/ | grep -i "<title>"`. Vérifier aussi qu'un rafraîchissement sur une page profonde ne renvoie pas 404 (§ 9, point 4).
- **Q16 – API.** La requête doit être en lecture seule et sans effet de bord, puisqu'elle tournera aussi en prod, avec un résultat vérifiable : code 200 et un champ attendu. C'est elle qui deviendra le Job `PostSync` (§ 7).
- **Q17 – Authentification.** Préférer que `/health/ready` fasse une vraie lecture dans Mongo, sans authentification. Sinon, un compte technique en lecture seule, dont le token est stocké comme les autres secrets.
- **Q18 – Mongo.** Demander aussi les index attendus, et vérifier après restauration qu'ils sont présents (`db.<collection>.getIndexes()`) : sans eux, l'appli fonctionne mais devient lente.

### À l'administrateur Kubernetes

| # | Question | Pourquoi | Réponse |
|---|---|---|---|
| 1 | Accepte-t-on une base de données dans le cluster ? Un opérateur Mongo est-il installé ? Sinon, Mongo hors cluster ? | Contenu du chart (§ 3.2) | |
| 2 | Version de Kubernetes, namespace dédié, quotas (ResourceQuota, LimitRange) ? | Cadre de déploiement | |
| 3 | Niveau de Pod Security (baseline, restricted) ? Moteur de politiques (Kyverno, Gatekeeper) et règles imposées ? | Non-root, limits, labels obligatoires | |
| 4 | Registries autorisés ? Qui fournit l'`imagePullSecret` Nexus ? | Tirage des images | |
| 5 | Quel ingress controller et quel `ingressClassName` ? Gateway API ? | Ingress (§ 9) | |
| 6 | Qui crée l'enregistrement DNS ? D'où viennent les certificats TLS (cert-manager, PKI interne) ? | Exposition | |
| 7 | Quelles StorageClass, avec quelle sauvegarde des volumes (Velero, snapshots) ? Agrandissement autorisé ? | Données Mongo | |
| 8 | NetworkPolicies en « deny » par défaut ? Proxy pour les sorties ? | Flux réseau | |
| 9 | Logs centralisés ? Métriques Prometheus, ServiceMonitor attendu ? | Exploitation | |
| 10 | PodDisruptionBudget attendu ? Fenêtres de maintenance ? | Drains de nœuds | |

**Recommandations, question par question**

- **Q1 – Base dans le cluster.** Position à défendre : en prod, Mongo hors cluster si une équipe DBA existe ; sinon l'opérateur MCK, installé et maintenu par l'admin ; le StatefulSet du chart reste réservé au dev. Si un opérateur existe, le chart remplace le StatefulSet par la ressource de l'opérateur (template à ajouter).
- **Q2 – Version et quotas.** Un namespace par environnement. Si un ResourceQuota existe, tout pod doit déclarer requests et limits (déjà le cas dans le chart) et la somme doit tenir dans le quota : demander les valeurs pour dimensionner les réplicas.
- **Q3 – Politiques.** Récupérer les politiques Kyverno ou Gatekeeper et les tester en local sur le rendu du chart (`helm template … > rendu.yaml`, puis `kyverno apply <politiques> --resource rendu.yaml`) : un refus détecté sur le poste coûte moins cher qu'un échec de synchronisation.
- **Q4 – Registries.** Si seul Nexus est autorisé, l'image Mongo doit aussi venir de Nexus (dépôt des images de référence) et non de Docker Hub. Faire créer l'`imagePullSecret` par l'admin à la création du namespace plutôt que de le gérer dans le chart.
- **Q5 – Ingress.** Si c'est ingress-nginx, demander le plan de migration (projet arrêté en mars 2026). Si les nouvelles applis passent en Gateway API, le chart aura besoin d'un template `HTTPRoute` à la place de l'Ingress.
- **Q6 – DNS et TLS.** Avec cert-manager et un émetteur interne, une annotation sur l'Ingress suffit (`cert-manager.io/cluster-issuer`) ; sinon, le certificat est fourni sous forme de Secret. Lancer la demande DNS tôt : c'est souvent le délai le plus long.
- **Q7 – Stockage.** Demander une StorageClass avec `reclaimPolicy: Retain` pour les bases : avec `Delete`, valeur par défaut fréquente, supprimer le PVC supprime les données. Le mode RWO suffit pour Mongo.
- **Q8 – NetworkPolicies.** Si tout est fermé par défaut, le chart doit ouvrir : contrôleur Ingress vers front et API, API vers Mongo, et tous les pods vers le DNS du cluster (port 53, souvent oublié). Template à ajouter au chart.
- **Q9 – Observabilité.** Si Prometheus Operator est en place, demander à Lilian si l'API expose `/metrics`, puis ajouter un ServiceMonitor au chart.
- **Q10 – PDB.** PodDisruptionBudget `minAvailable: 1` pour l'API et le front, qui ont 2 réplicas. **Pas** de PDB bloquant sur un Mongo à une seule instance : il empêcherait toute maintenance de nœud. Accepter une courte coupure, ou passer en replica set.

### À l'administrateur Argo CD

| # | Question | Pourquoi | Réponse |
|---|---|---|---|
| 1 | Qui crée l'Application et l'AppProject : nous, vous, ou un « app of apps » / ApplicationSet existant ? | Point d'entrée GitOps | |
| 2 | Quel mécanisme de secrets est branché (External Secrets avec Vault, Sealed Secrets, SOPS) ? | `existingSecret` (§ 3.3) | |
| 3 | Argo CD a-t-il accès à notre GitLab, et à Nexus si le chart y est publié ? | Sources du déploiement | |
| 4 | Politique de synchronisation imposée (sync automatique interdite en prod, fenêtres de sync) ? | `syncPolicy` | |
| 5 | Mise à jour des tags par commit de la CI, ou Argo CD Image Updater ? | Chaîne de livraison (§ 11) | |

**Recommandations, question par question**

- **Q1 – Application.** Demander un AppProject dédié, qui limite les dépôts sources et les namespaces de destination. S'il existe un « app of apps » ou un ApplicationSet standard, s'y brancher plutôt que de créer une Application isolée.
- **Q2 – Secrets.** External Secrets : ajouter au chart un template `ExternalSecret` qui pointe vers le chemin Vault et crée le Secret attendu par `existingSecret`. Sealed Secrets : chiffrer avec `kubeseal` et commiter le `SealedSecret`. SOPS : nécessite un plugin côté Argo CD. Dans tous les cas, le reste du chart ne change pas : il référence le nom du Secret.
- **Q3 – Accès.** Un jeton de déploiement **en lecture seule** pour Argo CD sur le dépôt de déploiement. Chart dans Git pour commencer : pas besoin d'un dépôt Helm dans Nexus tout de suite.
- **Q4 – Synchronisation.** Proposer : dev en automatique avec `prune` et `selfHeal` ; prod en manuel, ou en automatique avec fenêtres de synchronisation (§ 11).
- **Q5 – Tags.** Proposer le commit par la CI (§ 11) ; Image Updater seulement s'il est déjà installé et maintenu par l'admin.

---

## Sources

- [Bitnami – annonce des changements du catalogue (GitHub)](https://github.com/bitnami/containers/issues/83267)
- [Broadcom – préparer les changements Bitnami](https://community.broadcom.com/tanzu/blogs/beltran-rueda-borrego/2025/08/18/how-to-prepare-for-the-bitnami-changes-coming-soon)
- [MongoDB Controllers for Kubernetes – documentation](https://www.mongodb.com/docs/kubernetes-operator/)
- [MongoDB Controllers for Kubernetes – notes de version](https://www.mongodb.com/zh-cn/docs/kubernetes-operator/current/release-notes/)
- [Ingress NGINX Retirement: What You Need to Know (kubernetes.io)](https://kubernetes.io/blog/2025/11/11/ingress-Nginx-retirement/)
- [Ingress NGINX: Statement from the Kubernetes Steering and Security Response Committees](https://www.kubernetes.io/blog/2026/01/29/ingress-nginx-statement/)
