# ADR-001 : nommage des dépôts, chemins et routage Docker

- **Statut** : proposé, soumis à la communauté de pratique (CoP) pour approbation
- **Périmètre** : dépôts Nexus Repository Pro pour trois services et 60 applications
- **Remplace** : la convention générique de dépôts sans préfixe de service (`docker-candidat`, `raw-candidat`)
- **Décisions suivantes** : rôles et droits (ADR-002, ouvert)

---

## En bref (pour la CoP)

| Sujet | Décision proposée |
|---|---|
| Dépôts | `<service>-<format>-hosted` (candidat) et `<service>-<format>-release` |
| Chemin | `<application>/<composant>/<version>` : l'application en premier, sans le service |
| Promotion | Republication du candidat vers la release, même empreinte |
| Routage Docker | Un sous-domaine par dépôt Docker et par groupe Docker : 13 au total |
| Suppression sur `release` | Ouvert, traité dans ADR-002 |

**Ce que la CoP doit approuver** : les cinq lignes ci-dessus. Les exceptions de format (section 3) et le routage (section 5) demandent une attention particulière.

---

## 1. Contexte

Trois services, `oig`, `oigs` et `oigl`, d'environ 20 applications chacun, soit 60 applications. Chaque application utilise en moyenne 3 formats parmi npm, Maven, pypi, Docker et raw. Le volume est de 10 builds par jour au total. L'administration est identique sur les trois services. Les images de base et outils partagés (`mutualise`) servent aux trois services.

Une convention générique, sans préfixe de service, ne permet pas de régler les droits par service ni de lire un dépôt sans connaître l'ensemble du parc.

---

## 2. Décision : dépôts

| Type | Nom | Rôle |
|---|---|---|
| Candidat | `<service>-<format>-hosted` | Reçoit les publications de build |
| Release | `<service>-<format>-release` | Reçoit uniquement les versions promues, immuables |
| Mutualisé | `mutualise-<format>-hosted` et `mutualise-<format>-release` | Images dorées et outils partagés |
| Proxy | `proxy-<format>-<source>` | Relais des sources externes, partagé (ex. `proxy-npm-npmjs`) |
| Groupe par service | `<service>-<format>-all` | Lecture : release du service et proxys |
| Groupe mutualisé | `mutualise-<format>-all` | Lecture : release du mutualisé et proxys |

Règles :
- Le préfixe `<service>` vaut `oig`, `oigs` ou `oigl`. Le tiret après le préfixe est obligatoire : `oig-*` ne doit pas capturer `oigs-*` ni `oigl-*`.
- Le mot `hosted` désigne le candidat. Le runbook doit le dire, car le nom ne le montre pas.
- Le candidat n'entre jamais dans un groupe.
- Pas de qualificatif de variante. Aucune application n'en a besoin aujourd'hui.

---

## 3. Décision : chemins

Règle générale : `<application>/<composant>/<version>`. L'application est le premier segment. Le service n'apparaît pas dans le chemin, puisqu'il est dans le nom du dépôt.

Exceptions imposées par le format :

| Format | Forme | Exemple |
|---|---|---|
| npm | Portée = application ; fichier dans `-/` | `@diagb/web/-/web-1.2.0.tgz` |
| Maven | `groupId` = application | `diagb/diagb-api/1.2.0/` |
| Docker | `<application>/<composant>:<version>` | `diagb/image:1.2.0` |
| pypi | Dossier `packages/` et nom normalisé, gérés par Nexus | `packages/reporting-etl/2.1.0/…whl` |
| raw | Règle générale | `argo-library-base/config/1.4.0/config.tar` |

Le numéro de build (`-b142`) n'apparaît que dans la version candidat. La promotion ne change pas le chemin.

Les exemples complets sont dans `EXEMPLES.md`.

---

## 4. Décision : promotion

Par **republication** du candidat vers la release, avec copie de l'artefact, de son empreinte, de son manifeste et de son SBOM.

- Une version publiée en release n'est jamais modifiée.
- Le déploiement se fait par empreinte, jamais par tag.

Motif : conformité aux règles ANSSI et ReCyF sur l'immutabilité et la cohérence des artefacts déployés (voir `EXIGENCES/EX-22.md`).

---

## 5. Décision : routage Docker

Un **sous-domaine** par dépôt Docker et par groupe Docker, en TLS. Un DNS wildcard et un certificat wildcard couvrent l'ensemble. Le répartiteur route par nom d'hôte.

| Élément | Nombre | Sous-domaines |
|---|---|---|
| Dépôts Docker candidat, par service | 3 | 3 |
| Dépôts Docker release, par service | 3 | 3 |
| Dépôts Docker mutualisé (candidat et release) | 2 | 2 |
| Proxy Docker | 1 | 1 |
| Groupes Docker par service | 3 | 3 |
| Groupe Docker mutualisé | 1 | 1 |
| **Total** | | **13** |

**Pourquoi un sous-domaine plutôt qu'un chemin** : le protocole Docker attend le registre à la racine de son hôte. Un chemin (`<hôte>/docker-candidat/`) n'est pas accepté par les clients.

**Pourquoi un sous-domaine plutôt qu'un port** : un port par dépôt et par groupe multiplie les ports à ouvrir et à documenter, et complique le pare-feu. Le sous-domaine n'en demande qu'un.

Point à vérifier avant mise en service : le routage par nom d'hôte est-il pris en charge sur la version Nexus Pro retenue, groupes compris.

---

## 6. Options écartées

| Option | Raison du rejet |
|---|---|
| Convention générique (`docker-candidat`, sans service) | Ne sépare pas les trois services ; droits impossibles par service |
| Un dépôt par application | 360 à 480 dépôts hébergés, administration lourde, autant de sous-domaines Docker |
| Service dans le chemin | Redondant avec le nom du dépôt, sans gain |
| Mutualisé comme préfixe `mutualise/` dans un dépôt partagé | Mélange images dorées et artefacts applicatifs, complique les droits |
| Routage Docker par chemin | Non accepté par le protocole Docker |
| Promotion native Pro | Écartée au profit de la republication, pour la conformité (section 4) |

---

## 7. Critères de vérification

| Décision | Vérification |
|---|---|
| Nommage des dépôts | Liste des dépôts (API REST) conforme au tableau de la section 2 |
| Séparation par service | Un compte `oig` ne lit pas `oigs-*` ni `oigl-*` |
| Candidat hors groupe | Aucun groupe `*-all` ne contient de dépôt `hosted` |
| Release immuable | Une republication sur un chemin existant est rejetée (EX-22) |
| Promotion sans changement | Empreinte identique avant et après promotion |
| SBOM en release | Une recherche de composant retrouve les releases (EX-24) |
| Routage Docker | `docker login` réussit sur chacun des 13 sous-domaines |

---

## 8. Conséquences

- Les rôles et droits (ADR-002) reprennent ces noms de dépôts et ces préfixes d'application.
- Le DNS wildcard et le certificat wildcard doivent être prévus pour les 13 sous-domaines.
- Les clients Docker doivent faire confiance à l'autorité de certification interne.
- Le répartiteur doit être configuré pour router par nom d'hôte, avec une procédure dans le runbook.

---

## 9. Exigences liées

| Exigence | Type | Fiche | Usage dans cette ADR |
|---|---|---|---|
| EX-22 | Interne | [EX-22](EXIGENCES/EX-22.md) | Republication, immutabilité |
| EX-24 | Interne ; SSDF PS.3.2 | [EX-24](EXIGENCES/EX-24.md) | Copie du SBOM |
| EX-17 | Interne ; ReCyF 7.A.4 étendu | [EX-17](EXIGENCES/EX-17.md) | Reporté à ADR-002 |
| OF-22 | Recommandation ANSSI | [OF-22](EXIGENCES/OF-22.md) | Signature des tags : hors périmètre |
| OF-24 | ReCyF 5.A.1 | [OF-24](EXIGENCES/OF-24.md) | Cartographie, lien SBOM |
| OF-26 | ReCyF 5.B.8 | [OF-26](EXIGENCES/OF-26.md) | Proxys vers les sources officielles |
| OF-61 | NIST SSDF PS.2.1 | [OF-61](EXIGENCES/OF-61.md) | Empreinte accessible en release |

---

## 10. Points ouverts

- Suppression sur les dépôts `release` : traité dans ADR-002.
- Signature des tags et provenance (OF-22, SLSA) : ADR dédiée à créer.
- Routage par nom d'hôte sur Nexus Pro, groupes compris : à vérifier, procédure à noter.
- Affectation des 60 applications aux trois services : à compléter.
