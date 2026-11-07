# Story : Arborescence et nommage des repositories

Concaténation des sources factice liées à cette story (texte repris tel quel, sans réécriture). Sources : board, référentiel OF, ADR-0004, spec Nexus, dossier d'architecture, stories du lot 1.

## 1. Board : story 2 du socle Nexus

```
#### 2. Arborescence et nommage des repositories

**Exigences OF** : OF-02, OF-05, OF-26

Par service ou par domaine métier, avec les conventions de nommage, y compris pour les artefacts éditeur.
> **En tant qu'**équipe applicative,
> **je veux** savoir sans ambiguïté où publier mes artefacts et comment les nommer,
> **afin de** retrouver n'importe quelle version, la mienne ou celle d'une autre équipe, en quelques secondes.

**Critères OF** (proposition retenue dans le référentiel) :
- [ ] OF-02 : Séparation logique (VLAN, comptes, tags de runner) entre intégration et production ; physique pour la production si le RSSI la retient.
- [ ] OF-05 : Matrice de flux par brique, versionnée dans DocsSre/securite.
- [ ] OF-26 : Import initial depuis les sources officielles, puis distribution par Nexus uniquement.

```

## 2. Référentiel OF : lignes liées (OF-02, OF-05, OF-26)

### OF-02
- **ID** : OF-02
- **Thème** : Cloisonnement
- **Exigence (reformulation stricte du texte)** : Cloisonner les sous-systèmes entre eux, physiquement et/ou logiquement.
- **Texte source** : ANSSI, ReCyF v2.5 (document de travail, 17/03/2026)
- **Référence précise** : 7.A.4-EE
- **Objectif ReCyF** : Objectif 7
- **Nature** : Moyen acceptable de conformité ; l'objectif ReCyF associé devient obligatoire à la publication du décret (loi Résilience)
- **Applicable à** : EE
- **URL du texte** : https://messervices.cyber.gouv.fr/documents-ressources/20260317_NIS_V2_Suivi%20des%20modifications%20ReCyF%20v2.4%20vers%20v2.5-vfinale.pdf
- **Épique(s)** : Mise en service Nexus ; Infrastructure de build ; Inventaires et environnements
- **Impact AHRA et industrialisation** : Stories 4.1, 7 et 13 : runners, Nexus et secrets séparés entre intégration et production.
- **Proposition** : Séparation logique (VLAN, comptes, tags de runner) entre intégration et production ; physique pour la production si le RSSI la retient.
- **Alternatives** : Séparation logique seule (VLAN et IAM) ; séparation physique complète (instance Nexus et runners dédiés, retenue pour AHRA).

### OF-05
- **ID** : OF-05
- **Thème** : Filtrage
- **Exigence (reformulation stricte du texte)** : Définir et documenter les communications nécessaires entre chaque SI et les autres SI, et entre les sous-systèmes d'un SI.
- **Texte source** : ANSSI, ReCyF v2.5 (document de travail, 17/03/2026)
- **Référence précise** : 7.B.1-EI/EE ; 7.B.2-EE
- **Objectif ReCyF** : Objectif 7
- **Nature** : Moyen acceptable de conformité ; l'objectif ReCyF associé devient obligatoire à la publication du décret (loi Résilience)
- **Applicable à** : EI/EE (7.B.2 : EE)
- **URL du texte** : https://messervices.cyber.gouv.fr/documents-ressources/20260317_NIS_V2_Suivi%20des%20modifications%20ReCyF%20v2.4%20vers%20v2.5-vfinale.pdf
- **Épique(s)** : Mise en service Nexus ; Plateforme d'exécution
- **Impact AHRA et industrialisation** : Prérequis des stories 7 et 9 : aucune communication n'est encore documentée pour AHRA ni pour le POC.
- **Proposition** : Matrice de flux par brique, versionnée dans DocsSre/securite.
- **Alternatives** : Matrice dans l'outil réseau (CMDB NetBox) ; tableur non versionné (déconseillé).

### OF-26
- **ID** : OF-26
- **Thème** : Dépendances
- **Exigence (reformulation stricte du texte)** : Vérifier que toute nouvelle version d'un logiciel est téléchargée depuis les ressources officielles de l'éditeur ou du fournisseur.
- **Texte source** : ANSSI, ReCyF v2.5 (document de travail, 17/03/2026)
- **Référence précise** : 5.B.8-EI/EE
- **Objectif ReCyF** : Objectif 5
- **Nature** : Moyen acceptable de conformité ; l'objectif ReCyF associé devient obligatoire à la publication du décret (loi Résilience)
- **Applicable à** : EI/EE
- **URL du texte** : https://messervices.cyber.gouv.fr/documents-ressources/20260317_NIS_V2_Suivi%20des%20modifications%20ReCyF%20v2.4%20vers%20v2.5-vfinale.pdf
- **Épique(s)** : Classification des artefacts ; Mise en service Nexus
- **Impact AHRA et industrialisation** : Story 1 (classification) et story 5 : tout téléchargement d'éditeur passe par Nexus.
- **Proposition** : Import initial depuis les sources officielles, puis distribution par Nexus uniquement.
- **Alternatives** : Proxy Nexus vers dépôts publics pour l'import seulement ; miroir manuel.

## 3. ADR-0004 (intégral)

# ADR-0004 — Nommage des dépôts et RBAC du registry

- **Statut** : proposé (à accepter par vous)
- **Exigences liées** : REG-01 (nommage), REG-02 (RBAC) ; ACC-01 à ACC-04 et ACC-07
- **Spec** : `nexus-spec.md` §3.1 (matrice des dépôts), §3.2 (blob stores), §4 (identifiant et chemins), §5.1 (rôles)

## Contexte

Le lot 1 démarre avec un pilote, Achera (`service.hra`), géré à la main. Il faut fixer maintenant la structure des dépôts et les rôles, car ils conditionnent les droits, les chemins et les sélecteurs de contenu. Les règles de nomenclature complètes viendront plus tard.

## Décision

### Nommage des dépôts

- Un dépôt par format et par cycle de vie : `<format>-candidat`, `<format>-release` (exemple : `maven-candidat`, `maven-release`). Les noms sont en minuscules, sans suffixe de version.
- Un blob store par dépôt, avec quota souple.
- Les dépôts proxy portent le préfixe `proxy-` (exemple : `maven-proxy-central`).
- Le format exact d'Achera (Maven, Docker, raw) détermine les dépôts créés. Le lot 1 ne crée que les dépôts de ce format.

### Nommage des chemins

- L'identifiant `service.hra` est en tête du chemin de chaque artefact (Maven : `service/hra/…` ou groupId `service.hra.*` selon le format ; Docker : premier segment `service.hra/<image>`).
- Un sélecteur de contenu limite chaque application à son préfixe.

### Rôles (RBAC)

| Rôle | Périmètre | Droits |
|---|---|---|
| `lecteur-service.hra` | Dépôts release de `service.hra` | Lecture |
| `editeur-service.hra` | Dépôt candidat de `service.hra` | Lecture et publication en candidat |
| `promotion` | Compte de la chaîne de promotion | Promotion candidat vers release |
| `mutualise-editeur` | Images dorées | Publication (équipe qui construit les images) |
| `ssi-controle` | Transverse | Lecture, gestion des quarantaines et exceptions |
| `exploitation` | Administration technique | Administration, sans publication |

Règles :
- Moindre privilège : un compte n'a que les rôles de sa matrice (ACC-03).
- Séparation : `editeur-service.hra` ne promeut pas ; `promotion` ne publie pas (ACC-04).
- Comptes techniques : le compte de déploiement a le rôle `lecteur-service.hra` seulement (ACC-07).
- Comptes locaux au lot 1, groupes d'annuaire à mapper plus tard.

## Alternatives écartées

- **Un dépôt par application** : multiplie les dépôts (environ 60 applications dans la spec) et complique la promotion.
- **Un rôle global « développeur »** : ne respecte ni le moindre privilège ni la séparation des tâches.
- **Versionner le nom du dépôt** (`maven-candidat-2`) : contraire à REG-01 et à la règle de la spec §3.1.

## Points à confirmer

- **Format d'Achera** : sans lui, les dépôts et l'exemple Docker/Maven ne sont pas définitifs.
- **Dépôts proxy** : à créer au lot 1 ou plus tard ?
- **Membres des rôles** : qui détient `ssi-controle` et `exploitation` ?

## Conséquences

- Le nom des dépôts et les rôles sont figés pour le lot 1 ; un changement demande un nouvel ADR.
- Les règles de nomenclature complètes (identifiants, noms d'images) viendront dans un ADR ultérieur.

## 4. Spec Nexus : §3 Dépôts et §4 Convention de chemin

## 3. Dépôts

### 3.1 Matrice

| Format | Candidat | Release | Proxy | Groupe (lecture) |
|---|---|---|---|---|
| Maven | `maven-candidat` | `maven-release` | `maven-proxy-central` | `maven-all` |
| npm | `npm-candidat` | `npm-release` | `npm-proxy-npmjs` | `npm-all` |
| Docker | `docker-candidat` | `docker-release` | `docker-proxy-public` | `docker-all` |
| Raw | `raw-candidat` | `raw-release` | néant (les COTS sont déposés, pas proxifiés) | néant |

Règles :
- Les dépôts candidat et release sont de type hébergé. Les proxys ne sont jamais publiés vers.
- Les groupes exposent à la lecture l'ensemble release + proxy ; le candidat n'est jamais dans un groupe consommé par les déploiements.
- Le proxy Docker alimente uniquement la construction des images dorées (D6) ; aucune application ne tire directement depuis lui.

### 3.2 Paramétrage

| Paramètre | Candidat | Release | Proxy |
|---|---|---|---|
| Politique de redéploiement | Interdit | Interdit | Sans objet |
| Politique de version Maven | Release | Release | Release |
| Validation stricte du type de contenu | Activée | Activée | Activée |
| Blob store | Un par format et cycle de vie, avec quota souple | Idem | Idem |
| Nettoyage | Oui (3.4) | Non par défaut ; exception validée par la gouvernance | Oui (3.4) |
| Accès anonyme | Désactivé | Désactivé | Désactivé |

Le nommage évite tout suffixe de version dans le nom du dépôt. Une release n'est jamais supprimée par un nettoyage automatique.

### 3.3 Docker : point d'attention

Un dépôt Docker hébergé nécessite un point d'accès distinct (port dédié ou routage par sous-domaine, ou par chemin selon la version et le proxy inverse). Décision à figer en phase de conception détaillée : un sous-domaine par dépôt Docker, terminé sur le répartiteur de charge. Les noms d'images suivent la règle de chemin de la section 4.

### 3.4 Politiques de nettoyage

| Cible | Critère | Durée à fixer | Note |
|---|---|---|---|
| Candidats | Non promus et plus anciens que N jours | à fixer avec l'exploitation | Un candidat promu n'est pas supprimé tant que sa release existe |
| Caches proxy | Non téléchargés depuis N jours | à fixer | Évite la croissance libre du stockage |
| Images Docker candidates | Étiquettes anciennes sans référence de release | à fixer | Suivi du digest promu |
| Compactage des blobs | Tâche planifiée après nettoyage | Hebdomadaire | Libère réellement l'espace |

## 4. Convention de chemin et identifiants

### 4.1 Identifiant d'application

Format : `service.projet`, en minuscules, stable et unique (par exemple `monex.diagb`, `monex.jira`). Il est enregistré dans la CMDB, qui fait autorité. Aucune application ne publie sans identifiant enregistré.

### 4.2 Chemin par format

| Format | Emplacement de l'identifiant | Exemple de chemin résultant |
|---|---|---|
| Raw | Premier niveau du chemin | `monex.diagb/diagb/1.2/<paquet éditeur>` |
| Maven | Préfixe du groupId | groupId `monex.diagb.*`, soit chemin `monex/diagb/...` |
| npm | Portée (scope) dérivée de l'identifiant | portée `@monex-diagb` (points remplacés par un tiret, à confirmer) |
| Docker | Premier segment du nom d'image | `monex.diagb/<image>:<version>` |
| Mutualisé | Préfixe fixe `mutualise` | `mutualise/image-doree-temurin-21/<version>/` |

Règles raw :
- Chemin : `<identifiant>/<composant>/<version>/<fichier>`.
- Candidat : `<version>-<numéro de build>` pour les images dorées ; version éditeur inchangée pour les COTS (D2).
- Release : la version seule.

### 4.3 Étiquettes

| Étiquette | Contenu | Portée |
|---|---|---|
| `domaine` | Domaine métier | Recherche et tableaux de suivi |
| `equipe` | Équipe responsable | Recherche |
| `proprietaire` | Propriétaire de l'application | Recherche |
| `build` | Numéro de build de promotion | Lien candidat / release |

Le domaine, l'équipe et le propriétaire ne figurent jamais dans le chemin : ils changent, l'identifiant non.


## 5. Dossier d'architecture : décisions D4, D5 et annexes A.2 et A.3

| D4 | Le nom d'origine est conservé ; la normalisation se fait dans la CMDB | Normalisation en kebab-case à l'ingestion | Réécrire un nom modifie les coordonnées et invalide signatures et sommes de contrôle. |
| D5 | Un dépôt par format et par cycle de vie. L'identifiant d'application (service.projet) ouvre le chemin des dépôts hébergés ; ce qui est partagé va sous « mutualise ». Le domaine métier, l'équipe et le propriétaire passent par des étiquettes | Chemins par domaine métier ou par organigramme, avec double structure pendant la migration | Un domaine ou une équipe change ; l'identifiant d'application est stable et unique. Il permet de retrouver les artefacts d'une application, et de régler les droits par préfixe avec un sélecteur de contenu. Deux structures en parallèle doublent la maintenance. |

## Annexe A. Exemple : applications monex.diagb et monex.jira

Cet exemple illustre D5, le traitement des composants éditeur et des images dorées, et la manière de retrouver les éléments d'une application.

### A.1 Composition des applications

| Application | Composant | Serveur | Origine | Format dans Nexus | Identité |
|---|---|---|---|---|---|
| monex.diagb | diagb 1.2 | Application | Éditeur (COTS) | Raw | Nom d'origine, version, empreinte éditeur |
| monex.diagb | Temurin 21 | Application | Image dorée mutualisée | Raw | Version de l'image, commit de construction |
| monex.diagb | MongoDB 7.0 | Base de données | Image dorée mutualisée | Raw | Version de l'image, commit de construction |
| monex.diagb | MariaDB 11.x | Base de données | Image dorée mutualisée | Raw | Version de l'image, commit de construction |
| monex.jira | Jira Data Center 11.3 | Application | Éditeur (COTS) | Container | Digest, version, empreinte éditeur |
| monex.jira | Temurin 21 | Application | Image dorée mutualisée (la même que monex.diagb) | Raw | Version de l'image, commit de construction |
| monex.jira | PostgreSQL 17 | Base de données | Image dorée mutualisée (conteneur) | Docker | Digest, version, commit de construction |

### A.2 Arborescence Nexus

```text
Nexus
|
+-- raw-candidat            (hébergé, raw)
|   +-- monex.diagb/
|   |   +-- diagb/1.2/                          paquet éditeur
|   +-- mutualise/
|       +-- image-doree-temurin-21/<version>-<n° de build>/
|       +-- image-doree-mongodb-7.0/<version>-<n° de build>/
|       +-- image-doree-mariadb-11/<version>-<n° de build>/
|
+-- raw-release             (hébergé, raw, immuable)
|   +-- monex.diagb/
|   |   +-- diagb/1.2/                          paquet + SBOM (reconstitué)
|   +-- mutualise/
|       +-- image-doree-temurin-21/<version>/   image + SBOM + commit
|       +-- image-doree-mongodb-7.0/<version>/  image + SBOM + commit
|       +-- image-doree-mariadb-11/<version>/   image + SBOM + commit
|
+-- docker-candidat         (hébergé, Docker)
|   +-- monex.jira/jira-datacenter:<version>-<n° de build>
|   +-- mutualise/image-doree-postgresql-17:<version>-<n° de build>
|
+-- docker-release          (hébergé, Docker, immuable)
|   +-- monex.jira/jira-datacenter:<version>   digest + SBOM + commit
|   +-- mutualise/image-doree-postgresql-17:<version>   digest + SBOM + commit
|
+-- docker-proxy            (proxy du dépôt Docker public)
    +-- postgres 17         entrée de la construction de l'image dorée
```

### A.3 Retrouver les éléments d'une application

| Besoin | Chemin d'accès | Fiabilité |
|---|---|---|
| Parcourir tout ce qui est propre à une application | Le préfixe de l'application (monex.diagb) dans chaque dépôt hébergé | Directe |
| Retrouver toutes les applications d'un service | Le préfixe du service (monex.) dans chaque dépôt hébergé | Directe |
| Chercher par propriétaire, domaine ou équipe | Les étiquettes et la recherche Nexus | Bonne pour ce qui est hébergé |
| Voir tout ce qu'utilise une application, y compris le mutualisé | La CMDB : l'élément de l'application référence ses releases et les versions des images dorées | Référence de vérité |
| Savoir qui utilise une image dorée donnée | La CMDB : relations application vers version d'image | Référence de vérité |


## 6. Stories du lot 1 (intégral)

### [lot-1] S1 – Dépôts pour Achera

En tant que responsable plateforme, je veux des dépôts candidat et release pour les formats raw et Docker, afin de cadrer le stockage dès le départ.
- Les dépôts suivent la matrice de la spec (§3.1) pour raw et Docker.
- Création à la main selon un runbook. La même configuration doit pouvoir être reproduite plus tard par `roles/nexus`.
- Les dépôts release sont en politique d'écriture refusant tout remplacement (voir S7).

### [lot-1] S2 – Chemin et sélecteur de contenu `service.hra`

En tant que responsable de la gouvernance, je veux que tout artefact d'Achera soit sous le préfixe `service.hra`, afin qu'une application ne puisse publier que sous son identifiant (critère R4).
- Un sélecteur de contenu limite les dépôts Achera au préfixe, pour raw comme pour Docker (chemin selon la spec §4.2).
- Une publication sous un autre préfixe est refusée.

### [lot-1] S3 – Rôles et comptes (RBAC)

En tant que responsable de la sécurité, je veux les rôles de la spec (§5.1) en place, afin que chacun n'ait que les droits nécessaires.
- Rôles : `lecteur`, `editeur` (candidat `service.hra`), compte de promotion, `mutualise-editeur`, `ssi-controle`, `exploitation`.
- Un accès anonyme est refusé (critère R11).
- Comptes locaux tant que l'annuaire d'entreprise n'est pas connecté.

### [lot-1] S4 – Ingestion manuelle dans le candidat

En tant que développeur Achera, je veux téléverser un artefact et son SBOM dans le candidat selon un runbook, afin de tester le registry avant toute automatisation.
- Le chemin commence par `service.hra`.
- L'artefact est téléchargeable, avec son empreinte SHA-256.
- Le SBOM est joint à l'artefact (spec §6.5).

### [lot-1] S5 – Manifeste de release et attestations signées

En tant qu'auditeur, je veux un manifeste par artefact (version, empreinte, date, source, SBOM) et des attestations signées qui le lient à l'artefact, afin de vérifier ce qui est promu sans faire confiance à un document saisi à la main.
- Le manifeste suit les champs de la spec (§6.4).
- Le SBOM et la provenance sont attachés à l'artefact comme attestations, signées (signature keyless, type Cosign ou équivalent).
- La signature est vérifiée avant promotion (intégrée à S6).

### [lot-1] S6 – Promotion candidat vers release

En tant que responsable de la release, je veux promouvoir un artefact du candidat vers la release, afin qu'il soit figé.
- Docker : la promotion se fait par digest (les tags sont mutables). Raw : méthode selon ce que Nexus Pro supporte (point §12 de la spec, à vérifier).
- L'empreinte est identique avant et après la promotion.
- Une promotion sans manifeste ou sans SBOM échoue (R2).

### [lot-1] S7 – Immutabilité de la release

En tant que responsable de la release, je veux qu'un artefact publié en release ne puisse pas être remplacé, afin de garantir ce qui a été déployé.
- Une tentative de republication sur un chemin existant est refusée (R1).

### [lot-1] S8 – Supervision de base

En tant qu'exploitant, je veux une sonde de santé, afin de détecter une panne.
- La sonde de santé répond.

### [lot-1] S9 – Runbook d'exploitation

En tant qu'exploitant, je veux un runbook couvrant création de dépôt, ajout d'utilisateur et promotion, afin de travailler sans connaître Nexus par cœur.

### [lot-1] S10 – Provenance de la promotion (R3)

En tant que responsable de la release, je veux qu'une promotion échoue si le commit source n'appartient pas à une branche protégée, afin que seul du code relu soit promu.
- Un candidat dont le commit est inconnu ou hors branche protégée est refusé (R3).

## Idea (hors lot 1)

- **Pare-feu de dépôt / quarantaine à l'entrée** (spec §2, critère R5) : à traiter plus tard, avec la couverture du format raw à confirmer auprès de l'éditeur.
- **Sauvegarde et restauration** (spec §10.1, R10) : infrastructure, hors périmètre de ce lot.

## À confirmer avant de finaliser

- **Annuaire d'entreprise** : les comptes locaux suffisent-ils pour le lot 1 ?


## 7. Exemples

Les chemins ci-dessous viennent de l'annexe A.2 du dossier d'architecture et de la spec §4.2. Les contre-exemples se déduisent des règles de l'ADR-0004 et de la spec ; ils ne figurent pas tels quels dans les sources.

### 7.1 Applications de l'annexe A (monex.diagb et monex.jira)

| Application | Composant | Dépôt | Chemin |
|---|---|---|---|
| monex.diagb | diagb 1.2 (COTS, raw) | `raw-candidat` | `monex.diagb/diagb/1.2/` |
| monex.diagb | diagb 1.2 (COTS, raw) | `raw-release` (immuable, + SBOM reconstitué) | `monex.diagb/diagb/1.2/` |
| monex.diagb | Temurin 21 (image dorée) | `raw-release` | `mutualise/image-doree-temurin-21/<version>/` |
| monex.jira | Jira Data Center 11.3 (COTS, Docker) | `docker-candidat` | `monex.jira/jira-datacenter:<version>-<n° de build>` |
| monex.jira | Jira Data Center 11.3 (COTS, Docker) | `docker-release` (immuable, digest) | `monex.jira/jira-datacenter:<version>` |
| monex.jira | PostgreSQL 17 (image dorée) | `docker-release` | `mutualise/image-doree-postgresql-17:<version>` |

Retrouver ces éléments (annexe A.3) :
- Tout ce qui concerne `monex.diagb` : le préfixe `monex.diagb` dans chaque dépôt hébergé.
- Tout ce qui concerne le service `monex` : le préfixe `monex.` dans chaque dépôt hébergé.
- Tout ce qu'utilise `monex.jira`, mutualisé compris : la CMDB, qui est la référence de vérité.

Une mise à jour de l'image Temurin 21 touche `monex.diagb` et `monex.jira` : la CMDB le montre par la relation entre l'image et les applications.

### 7.2 Application de l'ADR-0004 (Achera, `service.hra`)

| Format | Dépôt | Chemin |
|---|---|---|
| Docker | `docker-candidat` | `service.hra/<image>:<version>-<n° de build>` |
| Docker | `docker-release` | `service.hra/<image>:<version>` |
| Maven | `maven-release` | groupId `service.hra.*`, chemin `service/hra/…` |
| Raw | `raw-release` | `service.hra/<composant>/<version>/<fichier>` |

Le format réel d'Achera reste à confirmer (point ouvert de l'ADR).

### 7.3 Contre-exemples (refusés par les règles)

| Ce qu'on voudrait faire | Pourquoi c'est refusé | Règle |
|---|---|---|
| Publier `maven-candidat-2` pour une nouvelle version du dépôt | Version dans le nom du dépôt | REG-01, spec §3.1 |
| Publier `monex.diagb/…` sous le préfixe `monex.jira` | Le sélecteur de contenu n'autorise que le préfixe de l'application | Lot 1 S2, critère R4 |
| Chemin `finance/diagb/1.2/` (domaine métier dans le chemin) | Le domaine change, l'identifiant non | Spec §4.3, D5 |
| Publier un composant sous un identifiant non enregistré | Aucune application ne publie sans identifiant enregistré dans la CMDB | Spec §4.1 |
| Renommer `Jira-DataCenter` en `jira-datacenter` à l'ingestion | Le nom d'origine est conservé, la normalisation se fait dans la CMDB | D4 |
| Republier `monex.jira/jira-datacenter:<version>` en release | Release immuable, redéploiement interdit | Spec §3.2, lot 1 S7 |
