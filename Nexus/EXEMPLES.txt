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

Le candidat a les mêmes chemins, dans le dépôt `<service>-<format>-hosted`, avec le numéro de build dans la version : `1.2.0-b142`.

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
