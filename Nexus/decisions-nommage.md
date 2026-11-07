# Nexus : décisions de nommage et d'arborescence

Services : `oig`, `oigs`, `oigl`. Environ 20 applications par service, soit 60 au total. Dépôts à partir de zéro, sans reprise de l'existant.

## Décisions

1. **Dépôts** : `<service>-<format>-hosted` pour le candidat, `<service>-<format>-release` pour la release.
   Exemples : `oig-npm-hosted`, `oigs-npm-release`, `oigl-docker-release`.
   Le mot `hosted` désigne le candidat : à documenter dans le runbook.
   Le tiret après le préfixe est obligatoire : `oig-*` ne doit pas capturer `oigs-*` ni `oigl-*`.

2. **Chemin** : `<application>/<composant>/<version>`. L'application est toujours le premier segment.

3. **Exceptions imposées par le format**
   - **npm** : portée = application (`@diagb/web`). Fichier dans `-/` : `@diagb/web/-/web-1.2.0.tgz`.
   - **Maven** : `groupId = diagb`, donc `diagb/diagb-api/1.2.0/`.
   - **Docker** : `<application>/<composant>:<version>`, sans service dans le chemin.
   - **pypi** : dossier `packages/` et nom normalisé, gérés par Nexus.

4. **Pas de service dans le chemin.** Le service n'est que dans le nom du dépôt.

5. **Candidat et release** : même chemin. Le numéro de build (`-b142`) n'apparaît que dans la version candidat. La promotion copie le fichier sans changer son chemin.

6. **Identifiant d'application figé à vie** (`service.app` pour la clé, la CMDB pour les attributs).

7. **Mutualisé** : `mutualise-<format>-hosted` (candidat) et `mutualise-<format>-release`. Même schéma que les services, `mutualise` tenant le rôle de service. Le chemin reste `<application>/…`.

8. **Proxys** : `proxy-<format>-<source>`, partagés par tous les services, sans service dans le nom. Exemples : `proxy-npm-npmjs`, `proxy-maven-central`, `proxy-docker-dockerhub`. Le préfixe distingue le proxy d'un dépôt `hosted`.

8 bis. **Groupes** : `<service>-<format>-all`, qui expose la release du service et les proxys partagés. Le candidat n'entre jamais dans un groupe.

9. **Qualificatif de variante** (ex. `chocolatey`) : règle à trancher. Critère retenu : adresse ou commande client différente, ou politique différente (rétention, droits, source) → dépôt séparé avec qualificatif avant le cycle, ex. `oig-nuget-chocolatey-release`. Aucune application n'en a besoin actuellement.

## Périmètre

- Formats : npm, Maven, pypi, Docker, raw.
- Formats par application : 3 en moyenne.
- Volume : 10 builds par jour au total.
- Dépôts hébergés : 3 services × 5 formats × 2 cycles = 30, plus 10 pour le mutualisé.

## Points ouverts

- Politique de suppression sur les dépôts `release` (procédure d'exception écrite).

## Exemples

Les noms d'applications et de services ci-dessous sont illustratifs. L'affectation des applications aux services `oig`, `oigs` et `oigl` reste à confirmer.

| Service | Application | Composant | Format | Dépôt release | Chemin |
|---|---|---|---|---|---|
| oigs | diagb | diagb-web | npm | oigs-npm-release | `@diagb/web/-/web-1.2.0.tgz` |
| oigs | diagb | diagb-api | Maven | oigs-maven-release | `diagb/diagb-api/1.2.0/` |
| oigs | diagb | diagb-image | Docker | oigs-docker-release | `diagb/image:1.2.0` |
| oigl | argo-library-base | execution-environment | Docker | oigl-docker-release | `argo-library-base/execution-environment:1.4.0` |
| oigl | argo-library-base | library-base-tools | pypi | oigl-pypi-release | `packages/argo-library-base-tools/0.3.0/argo_library_base_tools-0.3.0-py3-none-any.whl` |
| oigl | argo-library-base | config | raw | oigl-raw-release | `argo-library-base/config/1.4.0/config.tar` |
| oig | reporting | metabase | Docker | oig-docker-release | `reporting/metabase:0.50.0` |
| oig | reporting | reporting-etl | pypi | oig-pypi-release | `packages/reporting-etl/2.1.0/reporting_etl-2.1.0-py3-none-any.whl` |
| oig | reporting | dashboards | raw | oig-raw-release | `reporting/dashboards/2.1.0/dashboards.zip` |
