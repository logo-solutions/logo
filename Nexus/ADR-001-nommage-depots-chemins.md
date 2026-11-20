# ADR-001 : nommage des dépôts, chemins et routage Docker

- **Statut** : proposé, soumis à la communauté de pratique (CoP) pour approbation
- **Périmètre** : dépôts Nexus Repository Pro pour trois services (`oig`, `oigs`, `oigl`), regroupés sous une direction, soit 60 applications
- **Remplace** : la convention générique de dépôts sans préfixe (`docker-candidat`, `raw-candidat`), et la version précédente de cet ADR qui plaçait le service dans le nom des dépôts
- **Justification détaillée** : `nommage-depots-nexus.md`
- **Décisions suivantes** : rôles et droits (ADR-002, ouvert)

---

## En bref (pour la CoP)

| Sujet | Décision proposée |
|---|---|
| Dépôts | `<direction>-<format>-candidat` et `<direction>-<format>-release` |
| Périmètre | La direction, et non le service : un seul jeu de dépôts pour les trois services |
| Chemin | `<application>/<composant>/<version>` : l'application en premier |
| Promotion | Republication du candidat vers la release, même empreinte |
| Routage Docker | Un sous-domaine par dépôt Docker, en TLS, avec un DNS et un certificat wildcard |
| Suppression sur `release` | Ouvert, traité dans ADR-002 |

**Ce que la CoP doit approuver** : les six lignes ci-dessus. Le code de direction (section 2) et le routage Docker (section 5) demandent une attention particulière.

---

## 1. Contexte

Trois services, `oig`, `oigs` et `oigl`, d'environ 20 applications chacun, soit 60 applications, rattachés à une même direction. Chaque application utilise en moyenne 3 formats parmi npm, Maven, pypi, Docker, raw et yum. Le paquet RPM n'est pas un format distinct : il se dépose dans un dépôt `yum`. Le volume est de 10 builds par jour au total. L'administration Nexus est identique pour les trois services. Les images de base et outils partagés (`mutualise`) servent aux trois services.

Le service change avec les réorganisations, et Nexus ne sait pas renommer un dépôt. Mettre le service dans le nom impose donc de recréer les dépôts et de changer les adresses des clients à chaque réorganisation. Le nom d'application est déjà unique grâce à la CMDB. La direction, en revanche, est le niveau où les comptes, les droits et les règles de nettoyage changent : elle reste dans le nom des dépôts.

---

## 2. Décision : dépôts

| Type | Nom | Rôle |
|---|---|---|
| Candidat | `<direction>-<format>-candidat` | Reçoit les publications de build |
| Release | `<direction>-<format>-release` | Reçoit uniquement les versions promues, immuables |
| Mutualisé | `mutualise-<format>-<cycle>` | Images dorées et bibliothèques communes, en dépôts séparés |
| Proxy | `proxy-<format>-<source>` | Relais des sources externes, partagé (ex. `proxy-npm-npmjs`) |
| Groupe par direction | `<direction>-<format>-all` | Lecture : releases uniquement, sans les candidats |

Règles :
- Le préfixe `<direction>` est un **code de périmètre stable**, et non le nom affiché de la direction. Un renommage de la direction ne coûte alors rien ; une fusion ou une scission impose une migration. Le code reste à fixer avec le commanditaire.
- Le tiret après le préfixe est obligatoire : le code `dbx-*` ne doit pas capturer `dbxa-*`.
- La version est identique dans les deux dépôts : la promotion copie l'artefact sans le renommer.
- Le candidat n'entre jamais dans un groupe.
- Les ressources partagées (`mutualise`) sont dans des dépôts séparés, et non dans un chemin `mutualise/` : une image de base corrompue contamine les 60 applications, et ces dépôts ont leurs propres règles de conservation.
- Pas de qualificatif de variante. Aucune application n'en a besoin aujourd'hui.

---

## 3. Décision : chemins

Règle générale : `<application>/<composant>/<version>`. L'application est le premier segment. Le service n'apparaît ni dans le chemin ni dans le nom du dépôt : il reste dans la CMDB.

Ce choix tient aux coordonnées des artefacts : le nom d'une image (`dbx/api:1.4.2`) ou le scope d'un paquet npm (`@dbx/…`) ne change pas si l'application change de service. Le premier segment est le nom d'application de la CMDB, ce qui relie un artefact, son SBOM et ses vulnérabilités à l'application sans table de correspondance. Un préfixe suffit pour filtrer, et permet d'ajouter plus tard une vue par service sans déplacer d'artefact.

Exceptions imposées par le format :

| Format | Forme | Exemple |
|---|---|---|
| npm | Portée = application ; fichier dans `-/` | `@dbx/web/-/web-1.2.0.tgz` |
| Maven | `groupId` = application | `dbx/dbx-api/1.2.0/` |
| Docker | `<application>/<composant>:<version>` | `dbx/image:1.2.0` |
| pypi | Dossier `packages/` et nom normalisé, gérés par Nexus | `packages/reporting-etl/2.1.0/…whl` |
| raw | Règle générale | `argo-library-base/config/1.4.0/config.tar` |
| yum (RPM) | Racine du dépôt portée par `repodata/`, généré par Nexus ; chemin libre sous la racine | `argo-library-base/agent/1.0.0/agent-1.0.0.el8.x86_64.rpm` |

Le candidat et la release ont le **même chemin et la même version**, numéro de build compris (`dbx/image:1.2.0-b142`). La promotion change uniquement le dépôt (`candidat` vers `release`).

Les exemples complets sont dans `EXEMPLES.md`.

---

## 4. Décision : promotion

Par **republication** du candidat vers la release, avec copie de l'artefact, de son empreinte, de son manifeste et de son SBOM.

- Une version publiée en release n'est jamais modifiée.
- Le déploiement se fait par empreinte, jamais par tag.

Motif : conformité aux règles ANSSI et ReCyF sur l'immutabilité et la cohérence des artefacts déployés (voir `EXIGENCES/EX-22.md`).

---

## 5. Décision : routage Docker

Un **sous-domaine** par dépôt Docker, en TLS. Un DNS wildcard et un certificat wildcard couvrent l'ensemble. Le répartiteur route par nom d'hôte.

| Élément | Sous-domaines |
|---|---|
| Dépôts Docker de la direction : candidat et release | 2 |
| Dépôts Docker mutualisés : candidat et release | 2 |
| **Total dépôts Docker** | **4** |

Groupes et proxy Docker : **à confirmer**. Si chacun reçoit un sous-domaine, il faut ajouter 3 sous-domaines (groupe de la direction, groupe mutualisé, proxy), soit 7 au total.

**Pourquoi un sous-domaine plutôt qu'un chemin** : le protocole Docker attend le registre à la racine de son hôte. Un chemin (`<hôte>/<direction>-docker-candidat/`) n'est pas accepté par les clients.

**Pourquoi un sous-domaine plutôt qu'un port** : un port par dépôt multiplie les ports à ouvrir et à documenter, et complique le pare-feu. Le sous-domaine n'en demande qu'un.

**Contraintes de version**
- **Version minimale : Nexus Pro 3.69.** Le routage par sous-domaine y est disponible.
- **Routage par chemin : à réévaluer.** Il n'est disponible qu'à partir de la 3.83.0. Il supprimerait les sous-domaines, mais changerait les références d'images côté clients ; une période de transition serait alors nécessaire. Si la montée de version est proche, il vaut mieux attendre avant de passer les applications en Docker.
- **Noms en minuscules** : obligatoire pour les dépôts Docker depuis la 3.89.0. Nos noms le respectent déjà.
- **À vérifier avant mise en service** : le comportement des groupes Docker avec sous-domaine dédié sur la version Pro retenue, et les changements de push et de pull anonyme introduits en 3.88 et 3.98.

---

## 6. Options écartées

| Option | Raison du rejet |
|---|---|
| Convention générique (`docker-candidat`, sans direction) | Ne permet pas de lire un périmètre sans connaître l'ensemble du parc ni de régler les droits |
| Service dans le nom des dépôts (`oigs-npm-release`) | Migration des dépôts et changement d'adresse chez les clients à chaque réorganisation ; le service est déjà dans la CMDB ; sans gain de droits, la séparation n'étant pas exigée. Écarté après l'analyse de `nommage-depots-nexus.md` |
| Service dans le chemin | Le service entre dans les coordonnées des artefacts : à chaque réorganisation, les artefacts sont déplacés et les clients changent leurs références |
| Un dépôt par application | 360 à 480 dépôts hébergés, administration lourde, autant de sous-domaines Docker |
| Mutualisé comme préfixe `mutualise/` dans un dépôt partagé | Une erreur sur les images de base toucherait les 60 applications ; les règles de conservation ne peuvent pas différer |
| Routage Docker par chemin | Non accepté par le protocole Docker à la version actuelle |
| Promotion native Pro | Écartée au profit de la republication, pour la conformité (section 4) |

---

## 7. Critères de vérification

| Décision | Vérification |
|---|---|
| Nommage des dépôts | Liste des dépôts (API REST) conforme au tableau de la section 2 |
| Périmètre par direction | Un compte de publication de la direction ne publie que dans `<direction>-*` |
| Candidat hors groupe | Aucun groupe `*-all` ne contient de dépôt `candidat` |
| Release immuable | Une republication sur un chemin existant est rejetée (EX-22) |
| Promotion sans changement | Empreinte identique avant et après promotion |
| SBOM en release | Une recherche de composant retrouve les releases (EX-24) |
| Routage Docker | `docker login` réussit sur chacun des sous-domaines Docker déclarés |

---

## 8. Conséquences

- Les rôles et droits (ADR-002) reprennent ces noms de dépôts et ces préfixes d'application. Les comptes de service sont à définir par usage (publication, lecture), et non par service.
- Le DNS wildcard et le certificat wildcard doivent être prévus pour les sous-domaines Docker déclarés.
- Les clients Docker doivent faire confiance à l'autorité de certification interne.
- Le répartiteur doit être configuré pour router par nom d'hôte, avec une procédure dans le runbook.
- Une réorganisation qui fusionne ou scinde une direction impose une migration des dépôts. Un simple renommage de service n'a aucun effet sur Nexus.

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

- **Code de direction** : à fixer, stable, et à réserver sur l'instance Nexus.
- **Convention commune** : le RA Nexus n'en impose aucune. Lui proposer `<direction>-<format>-<cycle>` et lui faire réserver le préfixe, sinon une autre équipe peut créer un dépôt du même nom.
- **Lecture libre sur les 60 applications** : à faire valider par le commanditaire.
- **Dépôts séparés pour les ressources partagées** (`mutualise-*`) : à faire confirmer par le RA Nexus.
- **Sous-domaines Docker des groupes et du proxy** : à trancher (section 5).
- **Suppression sur les dépôts `release`** : traité dans ADR-002.
- **Signature des tags et provenance** (OF-22, SLSA) : ADR dédiée à créer.
- **Routage par nom d'hôte sur Nexus Pro, groupes compris** : à vérifier, procédure à noter.
