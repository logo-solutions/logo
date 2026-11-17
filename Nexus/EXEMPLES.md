# Exemples

Les noms d'applications et de services sont illustratifs. L'affectation réelle des 60 applications aux trois services `oig`, `oigs` et `oigl` reste à compléter.

## 1. Trois applications, un composant par format

| Service | Application | Composant | Format | Dépôt release | Chemin |
|---|---|---|---|---|---|
| oigs | diagb | diagb-web | npm | `oigs-npm-release` | `@diagb/web/-/web-1.2.0.tgz` |
| oigs | diagb | diagb-api | Maven | `oigs-maven-release` | `diagb/diagb-api/1.2.0/` |
| oigs | diagb | diagb-image | Docker | `oigs-docker-release` | `diagb/image:1.2.0` |
| oigl | argo-library-base | execution-environment | Docker | `oigl-docker-release` | `argo-library-base/execution-environment:1.4.0` |
| oigl | argo-library-base | library-base-tools | pypi | `oigl-pypi-release` | `packages/argo-library-base-tools/0.3.0/argo_library_base_tools-0.3.0-py3-none-any.whl` |
| oigl | argo-library-base | config | raw | `oigl-raw-release` | `argo-library-base/config/1.4.0/config.tar` |
| oig | reporting | metabase | Docker | `oig-docker-release` | `reporting/metabase:0.50.0` |
| oig | reporting | reporting-etl | pypi | `oig-pypi-release` | `packages/reporting-etl/2.1.0/reporting_etl-2.1.0-py3-none-any.whl` |
| oig | reporting | dashboards | raw | `oig-raw-release` | `reporting/dashboards/2.1.0/dashboards.zip` |
| oigl | argo-library-base | agent (RPM, illustratif) | yum | `oigl-yum-release` | `argo-library-base/agent/1.0.0/agent-1.0.0.el8.x86_64.rpm` |

Le candidat a les mêmes chemins et les mêmes versions, dans le dépôt `<service>-<format>-hosted`. Exemple : `diagb/image:1.2.0-b142` est présent à l'identique en candidat et en release.

## 2. Une application à trois tiers

Application `diagb`, service `oigs`.

| Tier | Composant | Format | Dépôt | Chemin |
|---|---|---|---|---|
| Présentation | diagb-web | npm | `oigs-npm-release` | `@diagb/web/-/web-1.2.0.tgz` |
| Applicatif | diagb-api | Maven | `oigs-maven-release` | `diagb/diagb-api/1.2.0/` |
| Applicatif (batch) | diagb-batch | pypi | `oigs-pypi-release` | `packages/diagb-batch/1.2.0/…whl` |
| Données | MongoDB 7.0 (image mutualisée) | raw | `mutualise-raw-release` | `image-doree-mongodb-7.0/7.0.14/` |
| Configuration | diagb-config | raw | `oigs-raw-release` | `diagb/config/1.2.0/config.tar` |

Le tier données est une image dorée mutualisée : elle est partagée par les trois services.

## 3. Une application COTS

Application `reporting`, service `oig`. Metabase est un produit éditeur, déposé et non construit.

| Composant | Origine | Format | Dépôt | Chemin |
|---|---|---|---|---|
| metabase | Éditeur (COTS) | Docker | `oig-docker-hosted`, puis `oig-docker-release` | `reporting/metabase:0.50.0` |

Pour un COTS, l'empreinte est celle publiée par l'éditeur. Le SBOM est reconstitué par analyse, l'éditeur n'en fournissant pas toujours.

## 4. Mutualisé

| Image | Format | Dépôt release | Chemin |
|---|---|---|---|
| Temurin 21 | raw | `mutualise-raw-release` | `image-doree-temurin-21/21.0.5/` |
| PostgreSQL 17 | Docker | `mutualise-docker-release` | `image-doree-postgresql-17:17.2` |

Les images dorées sont utilisées par les trois services. Elles sont publiées par l'équipe qui les construit, et lues par tous.

## 5. Routage Docker

Chaque dépôt Docker et chaque groupe Docker a son sous-domaine.

| Sous-domaine | Dépôt ou groupe |
|---|---|
| `oig-docker-hosted.<domaine>` | Candidat du service `oig` |
| `oig-docker-release.<domaine>` | Release du service `oig` |
| `oig-docker-all.<domaine>` | Groupe de lecture du service `oig` |
| `oigs-docker-hosted.<domaine>`, `oigs-docker-release.<domaine>`, `oigs-docker-all.<domaine>` | Idem pour `oigs` |
| `oigl-docker-hosted.<domaine>`, `oigl-docker-release.<domaine>`, `oigl-docker-all.<domaine>` | Idem pour `oigl` |
| `mutualise-docker-hosted.<domaine>`, `mutualise-docker-release.<domaine>`, `mutualise-docker-all.<domaine>` | Images dorées |
| `proxy-docker-dockerhub.<domaine>` | Relais Docker Hub |

Total : 13 sous-domaines. Un déploiement tire toujours depuis un groupe ou une release, jamais depuis le candidat.

## 6. Contre-exemples

Chaque cas refusé renvoie à une règle.

| Tentative | Refus | Règle |
|---|---|---|
| Publier `diagb/…` sous le préfixe de l'application `reporting` | Le sélecteur de contenu n'autorise que le préfixe de l'application | STORIES S2 |
| Republier une version existante en release | Redéploiement interdit | STORIES S6 |
| Déployer `image:latest` | Le tag est mutable : le déploiement se fait par empreinte | ADR-001, section 4 |
| Inclure le dépôt candidat dans un groupe | Le candidat n'entre jamais dans un groupe | ADR-001, section 2 |
| Nommer un dépôt `oig-npm-v2` | Le numéro de version n'est jamais dans le nom du dépôt | ADR-001, section 2 |

## 7. Application à deux tiers et outillage (schéma de déploiement)

Exemple tiré d'un schéma d'architecture existant. B2G-APP et B2G-BDD sont **deux tiers d'une même application**, `b2g`. Le nom du service reste à affecter (`<service>`).

Composants du schéma :

| Tier | Composant | Nature | Présent dans Nexus ? |
|---|---|---|---|
| Présentation / applicatif (B2G-APP) | Application d'entreprise, sur IIS | Livrable applicatif | Oui, format raw ou NuGet (voir point ouvert) |
| Présentation / applicatif (B2G-APP) | Microsoft .NET Framework 4.x | Composant Windows | Non : fonctionnalité du système, non publiée |
| Présentation / applicatif (B2G-APP) | Microsoft IIS 10 | Composant Windows | Non : fonctionnalité du système |
| Données (B2G-BDD) | Microsoft SQL Server 2022 Standard | Logiciel éditeur | Non publié dans Nexus : installé sur un master |
| Données (B2G-BDD) | Scripts de schéma et de données | Livrable applicatif | Oui, format raw |
| Outillage (hors application) | 7-Zip, Notepad++, Firefox ESR, Microsoft filterPack, Adobe PDF iFilter | COTS, éditeur | Oui, dépôt candidat raw, empreinte éditeur |

Chemins proposés, pour `<service>` :

| Dépôt release | Chemin | Contenu |
|---|---|---|
| `<service>-raw-release` | `b2g/app/<version>/` | Paquet applicatif (tiers présentation et applicatif) |
| `<service>-raw-release` | `b2g/bdd/scripts/<version>/` | Scripts SQL versionnés (tier données) |
| `<service>-raw-release` | `outillage/7-zip/<version>/7z-<version>.exe` | COTS, empreinte éditeur |

Le chemin commence par `b2g` : les deux tiers partagent le même premier segment, celui de l'application. L'outillage n'appartient à aucune application ; il est donc sous `outillage/`, en pseudo-application. Ce choix est à valider (voir ci-dessous).

Le candidat a les mêmes chemins dans `<service>-raw-hosted`, et les mêmes versions que la release.

**Points relevés par le schéma**

- Le flux F04 (déploiement d'une nouvelle version par SMB, non chiffré) ne respecte pas le principe de déploiement par le pipeline. Il est à remplacer par un flux chiffré et authentifié, tiré depuis la release par empreinte (ADR-001, section 4).
- Le flux F03 (accès à SQL Server en TLS, port 1433) est chiffré. Il n'est pas concerné par Nexus.
- Le format NuGet n'est pas dans le périmètre actuel. Si le livrable B2G-APP est un paquet .NET, il faut l'ajouter (ou le déposer en raw).

## 8. Tirer une image Docker depuis Nexus

Domaine fictif `nexus.entreprise.local`. Le jeton du compte de déploiement est lu depuis le coffre et n'apparaît jamais en clair.

**1. Connexion, avec le compte de déploiement (lecture seule sur la release)**

```bash
echo "$TOKEN_DEPLOIEMENT_OIGL" | docker login oigl-docker-all.nexus.entreprise.local \
  -u svc-deploy-oigl --password-stdin
```

**2. Tirage par étiquette, depuis le groupe de lecture**

```bash
docker pull oigl-docker-all.nexus.entreprise.local/argo-library-base/execution-environment:1.4.0
```

Le groupe expose la release de `oigl` et les proxys. C'est l'adresse à donner aux consommateurs.

**3. Tirage par digest, depuis la release : la référence de déploiement**

```bash
docker pull oigl-docker-release.nexus.entreprise.local/argo-library-base/execution-environment@sha256:<digest>
docker inspect --format '{{index .RepoDigests 0}}' oigl-docker-release.nexus.entreprise.local/argo-library-base/execution-environment:1.4.0
```

Le digest est lu après le premier tirage. Le déploiement référence ce digest, jamais l'étiquette `1.4.0`, qui reste mutable.

**4. Contrôle : le candidat est refusé au compte de déploiement**

```bash
docker pull oigl-docker-hosted.nexus.entreprise.local/argo-library-base/execution-environment:1.4.0-b142
# Résultat attendu : refus (401 ou 403). Le compte de déploiement ne lit que la release.
```

**Ce que cet exemple illustre**
- Un sous-domaine par dépôt et par groupe : le client ne spécifie jamais de chemin `/repository/...`.
- Le chemin de l'image (`argo-library-base/execution-environment`) suit ADR-001.
- Le déploiement passe par le digest, et le candidat reste inaccessible au déploiement (ADR-002, S3).
