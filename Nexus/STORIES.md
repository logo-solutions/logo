# Stories de mise en œuvre

Stories pour la mise en œuvre de Nexus Repository Pro, selon les décisions d'[ADR-001](ADR-001-nommage-depots-chemins.md) et les questions ouvertes d'[ADR-002](ADR-002-roles-droits.md).

Chaque story a : un besoin, des critères d'acceptation, une vérification, et les exigences liées. Le statut est « à faire » sauf mention contraire.

---

## S1 : dépôts par service et par format

**En tant que** responsable plateforme, **je veux** les dépôts candidat et release de chaque service et de chaque format, **afin de** publier et promouvoir sans dépôt partagé entre services.

**Critères d'acceptation**
- Pour chacun des 3 services (`oig`, `oigs`, `oigl`) et chacun des 5 formats (npm, Maven, pypi, Docker, raw) : un dépôt `hosted` et un dépôt `release`, soit 30 dépôts.
- Le mutualisé a ses dépôts propres : 10 dépôts.
- Chaque dépôt `release` est en politique « Disable redeploy ».
- Le nom de chaque dépôt suit ADR-001, section 2.

**Vérification** : liste des dépôts via l'API REST, comparée au tableau de ADR-001.

**Exigences liées** : EX-22, OF-22.

---

## S2 : chemin et sélecteur de contenu par application

**En tant que** responsable de la gouvernance, **je veux** qu'une application ne puisse publier que sous son préfixe, **afin que** deux applications ne s'écrasent pas.

**Critères d'acceptation**
- Un sélecteur de contenu par application, sur le préfixe `/<application>/`.
- Une publication sous un autre préfixe est refusée (code 403).
- Le chemin commence toujours par l'application, sauf exceptions de format (ADR-001, section 3).

**Vérification** : publication de test sous le bon préfixe (accepté), puis sous un autre (refusé).

**Exigences liées** : OF-10, OF-34.

---

## S3 : rôles et comptes de service

**En tant que** responsable de la sécurité, **je veux** des comptes de service qui n'ont que les droits de leur usage, **afin que** chaque pipeline agisse dans son périmètre.

**Dépend de** : ADR-002, points 2 et 3.

**Critères d'acceptation**
- Un compte de publication candidat ne peut pas écrire en release.
- Un compte de promotion ne peut pas écrire en candidat.
- Un compte de déploiement ne lit que la release.
- Un accès anonyme est refusé sur tous les dépôts.

**Vérification** : une tentative par combinaison compte / action, dont le résultat attendu est tracé.

**Exigences liées** : OF-10, OF-12, OF-13, EX-17.

---

## S4 : ingestion manuelle dans le candidat

**En tant que** développeur, **je veux** téléverser un artefact et son SBOM dans le candidat selon un runbook, **afin de** tester le dépôt avant l'automatisation.

**Critères d'acceptation**
- Le chemin commence par le nom de l'application.
- L'artefact est téléchargeable, avec son empreinte SHA-256.
- Le SBOM est joint à l'artefact.

**Vérification** : téléchargement, recalcul de l'empreinte, comparaison avec l'empreinte annoncée.

**Exigences liées** : EX-24, OF-61.

---

## S5 : republication du candidat vers la release

**En tant que** responsable de la release, **je veux** promouvoir un artefact du candidat vers la release, **afin de** figer ce qui a été contrôlé.

**Critères d'acceptation**
- La promotion copie l'artefact, son manifeste et son SBOM.
- L'empreinte est identique avant et après promotion.
- Une promotion sans SBOM échoue.
- Une promotion sans manifeste échoue.
- Pour Docker, la promotion se fait par digest.

**Vérification** : promotion d'un candidat complet (accepté), puis d'un candidat sans SBOM (refusé), puis comparaison des empreintes.

**Exigences liées** : EX-22, EX-24, OF-61.

---

## S6 : immutabilité de la release

**En tant que** responsable de la release, **je veux** qu'un artefact publié ne puisse pas être remplacé, **afin de** garantir ce qui a été déployé.

**Critères d'acceptation**
- Une republication sur un chemin existant en release est rejetée.

**Vérification** : tentative de republication, résultat rejeté, empreintes comparées.

**Exigences liées** : EX-22.

---

## S7 : routage Docker

**En tant que** développeur, **je veux** un point d'accès dédié pour chaque dépôt et groupe Docker, **afin que** les clients Docker fonctionnent sans préfixe de chemin.

**Critères d'acceptation**
- 13 sous-domaines résolus par le DNS wildcard.
- Un certificat wildcard couvre les 13 sous-domaines, émis par l'autorité interne.
- Un `docker login` réussit sur chacun.
- Le répartiteur route par nom d'hôte.

**Vérification** : `docker login` sur chacun des 13 sous-domaines.

**Point à vérifier** : la prise en charge du routage par nom d'hôte sur Nexus Pro, groupes compris (ADR-001, section 5).

---

## S8 : promotion Docker par digest

**En tant que** responsable de la release, **je veux** que chaque image promue soit identifiée par son digest, **afin que** le déploiement ne dépende d'aucun tag mutable.

**Critères d'acceptation**
- Le déploiement référence l'image par `sha256:…`.
- Aucun déploiement ne référence un tag `latest`.

**Vérification** : revue du manifeste de déploiement ; recherche de `latest` dans les déploiements.

**Exigences liées** : EX-22, OF-61.

---

## S9 : groupes de lecture

**En tant que** consommateur, **je veux** une adresse unique par service et par format, **afin de** lire les releases et les proxys sans connaître chaque dépôt.

**Critères d'acceptation**
- Un groupe `<service>-<format>-all` par service et format.
- Le groupe contient la release du service et les proxys partagés.
- Le candidat n'appartient à aucun groupe.
- Un groupe `mutualise-<format>-all` pour le mutualisé.

**Vérification** : configuration de chaque groupe ; lecture réussie depuis le groupe, échec depuis le candidat.

---

## S10 : proxys vers les sources officielles

**En tant que** responsable sécurité, **je veux** que les dépendances externes ne passent que par des proxys vers les sources officielles, **afin de** ne pas télécharger de logiciel hors de sources reconnues.

**Critères d'acceptation**
- Un proxy par source : `proxy-<format>-<source>`.
- Aucun service ne publie vers un proxy.
- Les COTS sont déposés dans le candidat à partir de l'éditeur, sans proxy.

**Vérification** : publication vers un proxy refusée ; téléchargement d'une dépendance via le proxy réussi.

**Exigences liées** : OF-26.

---

## S11 : mutualisé

**En tant que** responsable des images dorées, **je veux** publier les images partagées dans des dépôts dédiés, **afin que** les trois services les lisent sans les publier.

**Critères d'acceptation**
- Les dépôts `mutualise-<format>-hosted` et `mutualise-<format>-release` existent.
- Seule l'équipe qui construit les images peut publier dans `mutualise-*`.
- Les trois services peuvent lire `mutualise-<format>-release`.

**Dépend de** : ADR-002, point 6 (lecture inter-services).

---

## S12 : runbook d'exploitation

**En tant qu'** exploitant, **je veux** un runbook couvrant la création d'un dépôt, l'ajout d'un utilisateur, la promotion et le routage, **afin de** travailler sans connaître Nexus par cœur.

**Critères d'acceptation**
- Le runbook explique que `hosted` désigne le candidat.
- Le runbook décrit la création d'un dépôt, d'un sélecteur et d'un groupe.
- Le runbook décrit la configuration du répartiteur pour le routage Docker.

**Vérification** : relecture par une personne qui n'a pas participé à la rédaction.

---

## S13 : nettoyage du candidat

**En tant que** responsable plateforme, **je veux** purger les candidats anciens et non promus, **afin de** maîtriser le stockage sans jamais toucher à la release.

**Critères d'acceptation** (à confirmer)
- Durée de conservation du candidat : à fixer avec l'exploitation.
- Un candidat promu n'est pas supprimé tant que sa release existe.
- La release n'est jamais purgée par cette règle.

**Dépend de** : décision de conservation à prendre.

---

## S14 : suppression sur les releases

**En tant que** responsable de la release, **je veux** une règle écrite pour supprimer une version publiée, **afin de** corriger une erreur sans affaiblir l'immutabilité.

**Critères d'acceptation** : à définir après la décision d'ADR-002, point 1.

**Dépend de** : ADR-002, point 1.

---

## Ordre de mise en œuvre

1. S1 (dépôts) et S2 (sélecteurs)
2. S3 (comptes), après ADR-002
3. S4 (ingestion), S5 (republication), S6 (immutabilité)
4. S7 (routage Docker) et S8 (digest)
5. S9 (groupes), S10 (proxys), S11 (mutualisé)
6. S12 (runbook), S13 et S14 (après décisions)
