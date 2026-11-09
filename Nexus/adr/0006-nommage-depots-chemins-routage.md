# ADR-0006 : nommage des dépôts, règle de chemin et routage Docker

- **Statut** : proposé, en attente d'acceptation
- **Remplace** : ADR-0004 de factice (`DocsSre/fabrication/nexus/adr/0004-nommage-depots-et-rbac.md`), pour la partie nommage. Les rôles RBAC font l'objet d'une ADR suivante.
- **Édition** : Nexus Repository Pro (ADR-0001 de factice, accepté). Haute disponibilité hors périmètre (ADR-0003 de factice, non retenue ici).
- **Exigences** : voir la section « Exigences » et les fiches dans `Nexus/exigences/`.
- **Décisions détaillées** : `Nexus/decisions-nommage.md`.

## Contexte

Le client exploite trois services, `oig`, `oigs` et `oigl`, d'environ 20 applications chacun, soit 60 applications. Chaque application utilise en moyenne 3 formats parmi npm, Maven, pypi, Docker et raw. Le volume est faible : 10 builds par jour au total. L'administration est identique sur les trois services. Les images de base et les outils partagés (`mutualise`) servent aux trois services.

La spec de factice décrit un schéma générique (`docker-candidat`, `raw-candidat`, sans préfixe de service) et une règle de chemin `service.projet/…`. Ce schéma ne répond pas à la structure à trois services.

## Décision

### 1. Dépôts

| Type | Nom | Rôle |
|---|---|---|
| Candidat | `<service>-<format>-hosted` | Reçoit les publications de build |
| Release | `<service>-<format>-release` | Reçoit uniquement les versions promues, immuables |
| Mutualisé | `mutualise-<format>-hosted` et `mutualise-<format>-release` | Images dorées et outils partagés |
| Proxy | `proxy-<format>-<source>` | Relais des sources externes, partagé |
| Groupe par service | `<service>-<format>-all` | Lecture : release du service + proxys |
| Groupe mutualisé | `mutualise-<format>-all` | Lecture : release du mutualisé + proxys |

- Le préfixe `<service>` vaut `oig`, `oigs` ou `oigl`. Le tiret après le préfixe est obligatoire : `oig-*` ne doit pas capturer `oigs-*` ni `oigl-*`.
- Le mot `hosted` désigne le candidat. Il est à documenter dans le runbook, parce que le nom ne le dit pas.
- Le candidat n'entre jamais dans un groupe.
- Aucun qualificatif de variante n'est retenu. Aucune application n'en a besoin aujourd'hui.

### 2. Chemin

- Règle générale : `<application>/<composant>/<version>`. L'application est le premier segment. Le service n'apparaît pas dans le chemin.
- Exceptions imposées par le format :

| Format | Forme | Exemple |
|---|---|---|
| npm | Portée = application, fichier dans `-/` | `@diagb/web/-/web-1.2.0.tgz` |
| Maven | `groupId` = application | `diagb/diagb-api/1.2.0/` |
| Docker | `<application>/<composant>:<version>` | `diagb/image:1.2.0` |
| pypi | Dossier `packages/` et nom normalisé, gérés par Nexus | `packages/reporting-etl/2.1.0/…whl` |
| raw | Règle générale | `argo-library-base/config/1.4.0/config.tar` |

- Le numéro de build (`-b142`) n'apparaît que dans la version candidat. La promotion ne change pas le chemin.

### 3. Promotion

Par **republication** du candidat vers la release, avec copie du manifeste, du SBOM et de l'empreinte. Une version publiée en release n'est jamais modifiée. Le déploiement se fait par empreinte, jamais par tag.

Raison : conformité aux règles ANSSI et ReCyF (voir EX-22).

### 4. Routage Docker

Un **sous-domaine** par dépôt Docker et par groupe Docker, en TLS, selon l'option A de l'ADR-0005 de factice (encore proposée, à confirmer sur Nexus Pro).

| Élément | Nombre | Sous-domaines |
|---|---|---|
| Dépôts Docker `hosted` (candidat), par service | 3 | 3 |
| Dépôts Docker `release`, par service | 3 | 3 |
| Dépôts Docker mutualisé (`hosted` et `release`) | 2 | 2 |
| Proxy Docker (`proxy-docker-dockerhub`) | 1 | 1 |
| Groupes Docker par service (`<service>-docker-all`) | 3 | 3 |
| Groupe Docker mutualisé (`mutualise-docker-all`) | 1 | 1 |
| **Total** | | **13** |

Un DNS wildcard et un certificat wildcard couvrent les 13 sous-domaines. Le répartiteur route par nom d'hôte.

## Options écartées

- **Schéma générique de factice** (`docker-candidat`, `raw-candidat`, sans service) : ne sépare pas les trois services et ne permet pas de régler les droits par service.
- **Un dépôt par application** : 360 à 480 dépôts hébergés, administration lourde, autant de sous-domaines Docker.
- **Service dans le chemin** : redondant avec le nom du dépôt. Écarté, sauf quand le format impose son chemin.
- **Mutualisé comme préfixe `mutualise/` dans un dépôt partagé** (spec de factice, §4.2) : mélange les images dorées et les artefacts applicatifs, et complique les droits.
- **Routage Docker par chemin** : non compatible avec le protocole Docker, qui attend le registre à la racine de son hôte (ADR-0005 de factice, option C).
- **Promotion native Pro** : écartée pour l'instant, au profit de la republication (conformité, décision 3).

## Critères de vérification

| Décision | Vérification |
|---|---|
| Nom des dépôts | Liste des dépôts (`GET /service/rest/v1/repositories`) conforme au tableau |
| Séparation par service | Un compte `oig` ne lit pas `oigs-*` ni `oigl-*` |
| Candidat hors groupe | Le groupe `<service>-<format>-all` ne contient pas le `hosted` |
| Release immuable | Une republication sur un chemin existant est rejetée (EX-22) |
| Promotion sans changement | Empreinte identique avant et après promotion |
| SBOM en release | Une recherche de composant retrouve les releases (EX-24) |
| Routage Docker | Un `docker login` réussit sur chacun des 13 sous-domaines |

## Conséquences

- La spec §3.1 (matrice), §4 (chemins) et §5 (rôles) de factice ne s'appliquent pas à ce périmètre. Elles sont réécrites dans `Nexus/`, sans modifier factice.
- Les rôles RBAC (ADR suivante) reprennent ces noms de dépôts et ces préfixes d'application. Ils intégreront EX-17.
- Le client Docker doit faire confiance à l'autorité de certification interne.

## Exigences

| Exigence | Type | Fiche | Usage dans cette ADR |
|---|---|---|---|
| EX-22 | Interne | [EX-22](../exigences/EX-22.md) | Republication, immutabilité |
| EX-24 | Interne, SSDF PS.3.2 | [EX-24](../exigences/EX-24.md) | Copie du SBOM |
| EX-17 | Interne, ReCyF 7.A.4 extrapolé | [EX-17](../exigences/EX-17.md) | Reporté à l'ADR RBAC |
| OF-22 | ANSSI DevSecOps | [OF-22](../exigences/OF-22.md) | Signature des tags : hors périmètre |
| OF-24 | ReCyF 5.A.1 | [OF-24](../exigences/OF-24.md) | Cartographie, lien SBOM |
| OF-26 | ReCyF 5.B.8 | [OF-26](../exigences/OF-26.md) | Proxys vers les sources officielles |
| OF-61 | NIST SSDF PS.2.1 | [OF-61](../exigences/OF-61.md) | Empreinte accessible en release |

## Points ouverts

- **Politique de suppression sur les dépôts `release`** : ouvert. Le CSV de factice marque EX-22 comme exigence interne ; la règle de suppression doit donc être décidée, pas déduite d'une source.
- **Signature des tags et provenance** (OF-22, SLSA) : ADR dédiée.
- **Routage par nom d'hôte sur Nexus Pro**, y compris pour les groupes : à vérifier, et procédure à noter dans le runbook.
- **Affectation des 60 applications** aux trois services : à compléter.
- **Rôles RBAC** : ADR suivante, avec EX-17.
