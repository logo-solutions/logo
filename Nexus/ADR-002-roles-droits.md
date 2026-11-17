# ADR-002 : comptes de service, rôles et droits

- **Statut** : proposé, soumis à la communauté de pratique (CoP). Décisions prises en section 7.
- **Dépend de** : ADR-001 (noms de dépôts, préfixes d'application, groupes, nommage des comptes)
- **Exigences** : OF-10, OF-12, OF-13, OF-33, OF-34, OF-35, EX-17, EX-22 (voir `EXIGENCES/`)

---

## En bref (pour la CoP)

| Sujet | Décision proposée |
|---|---|
| Comptes | Un compte par service et par usage : `svc-<service>-<usage>`, soit au plus 12 comptes |
| Usages | `publication`, `promotion`, `deploiement`, `lecture` |
| Propriétaire | Une personne nommée par compte, avec un suppléant (option 1) |
| Jetons | Générés à la demande, durée limitée, stockés dans le coffre, jamais en clair |
| Séparation | Le propriétaire de `promotion` ne peut pas être celui de `publication` |
| Suppression sur `release` | Interdite à tous ; exception par procédure écrite |
| Droits par application | Un sélecteur de contenu par application, regroupé dans le rôle du service |
| Revue | Annuelle, et à chaque changement de propriétaire ou de droits |

**Ce que la CoP doit approuver** : les lignes ci-dessus, et les trois points ouverts de la section 7.

---

## 1. Contexte

ADR-001 fixe les dépôts par service (`<service>-<format>-hosted` et `-release`), les groupes de lecture et le nommage des comptes. Il reste à définir ce que chaque compte peut faire, qui en répond, et comment ses jetons sont gérés.

Les pipelines n'ont pas de compte administrateur. Les accès humains sont déjà protégés par SSO et MFA : ce document ne traite que des comptes de service.

---

## 2. Décision : comptes et droits

| Compte | Dépôts en écriture | Dépôts en lecture | Rôle Nexus |
|---|---|---|---|
| `svc-<service>-publication` | `<service>-<format>-hosted` | `<service>-<format>-hosted` | Publication en candidat |
| `svc-<service>-promotion` | `<service>-<format>-release` | `<service>-<format>-hosted` | Promotion du candidat vers la release |
| `svc-<service>-deploiement` | Aucun | `<service>-<format>-release`, via le groupe `<service>-<format>-all` | Lecture de la release seulement |
| `svc-<service>-lecture` | Aucun | `<service>-<format>-all` | Lecture de la release, pour l'exploitation |

Règles :
- **Aucun compte ne lit le candidat, sauf `publication` et `promotion`.** Le déploiement ne tire jamais depuis le candidat (ADR-001, section 2 ; S3).
- **Aucun compte n'écrit dans la release, sauf `promotion`.**
- **Le candidat n'entre dans aucun groupe** (ADR-001).
- **Le mutualisé suit le même schéma** avec `mutualise-<format>-*`. Seule l'équipe qui construit les images a le droit de publier dans `mutualise-*-hosted`.
- **Les droits sont portés par rôle**, par dépôt, et restreints au préfixe d'application lorsque c'est possible (ADR-001, section 3).

---

## 3. Décision : propriétaires

- **Un propriétaire nommé par compte**, et un suppléant qui peut agir en son absence.
- Le propriétaire répond de la création, de la revue, de la rotation des jetons et de la suppression du compte.
- **Séparation des tâches** : le propriétaire de `promotion` ne peut pas être le propriétaire de `publication` du même service.
- Le registre des comptes (propriétaire, suppléant, usage, date de création, date de dernière revue) est tenu dans la documentation de l'exploitation.

| Compte | Propriétaire | Suppléant |
|---|---|---|
| `svc-<service>-publication` | À nommer | À nommer |
| `svc-<service>-promotion` | À nommer (distinct de `publication`) | À nommer |
| `svc-<service>-deploiement` | À nommer | À nommer |
| `svc-<service>-lecture` | À nommer | À nommer |

---

## 4. Décision : jetons

- Les jetons sont **générés à la demande**, avec une durée de vie limitée configurée dans Nexus (version 3.69 ou plus, voir ADR-001).
- Le jeton est stocké **dans le coffre**, jamais dans un dépôt, ni dans une variable de CI en clair.
- La durée proposée est **90 jours**, à valider.
- Un jeton est révoqué immédiatement si le propriétaire change, si l'usage disparaît, ou si une fuite est suspectée.

---

## 5. Décision : revue

- **Annuelle** : propriétaire, usage et droits de chaque compte (OF-35).
- **À chaque changement** de propriétaire, de droits ou de périmètre.
- La revue est tracée dans le registre, avec sa date.

---

## 6. Pourquoi ce choix

- **Un compte par usage** limite ce qu'un jeton compromis peut faire (OF-34).
- **Un compte par service** garde le nombre de comptes à maintenir faible : au plus 12.
- **Un propriétaire nommé** évite les comptes orphelins (OF-35).
- **Une identité de charge (OIDC depuis la CI)** reste une trajectoire possible : elle supprimerait les jetons stockés. Son support par Nexus Pro est à vérifier avant d'être retenue.

---

## 7. Décisions prises et point ouvert

1. **Suppression sur les dépôts `release` : interdite.** Aucun compte, y compris l'exploitation, ne peut supprimer une version publiée. Une correction passe par une nouvelle version, et une exception reste possible uniquement par procédure écrite à deux validations.
2. **Droits par application : un sélecteur par application.** Le service est dans le nom du dépôt, et le chemin commence par l'application (ADR-001). Les applications d'un service n'ont donc pas de préfixe commun : le compte `publication` reçoit un sélecteur par application, regroupé dans son rôle (environ 20 par service).
3. **Runners et comptes (EX-17) : trois runners distincts**, avec des tags distincts :
   - **publication** : build et publication dans le candidat, compte `svc-<service>-publication` ;
   - **promotion** : copie du candidat vers la release, compte `svc-<service>-promotion` ;
   - **déploiement** : lecture de la release et déploiement, compte `svc-<service>-deploiement`.
   Interprétation à valider : l'extrapolation depuis ReCyF 7.A.4 n'est pas un texte direct.

Points à traiter ailleurs :
- **Administrateurs humains** : qui détient le rôle `exploitation`, et faut-il un rôle `ssi-controle`.
- **Lecture inter-services** : un service peut-il lire les releases d'un autre ?

---

## 8. Conséquences

- Les noms des comptes et des rôles suivent ADR-001.
- Le registre des comptes est un document à tenir à jour, relu chaque année.
- Le coffre doit être prévu pour distribuer les jetons aux pipelines.
- L'accès anonyme reste désactivé sur tous les dépôts (spec, §3.2).

---

## Annexe A. État de l'art et alternatives étudiées

### A.1 État de l'art

**Exigences vérifiées** :
- **OF-10** (ANSSI, DevSecOps) : moindre privilège pour la CI/CD de production, et jetons « générés à la demande ».
- **OF-33** (ReCyF) : comptes individuels pour les processus automatiques, chacun réservé à son usage.
- **OF-34** (ReCyF) : uniquement les droits nécessaires, sur chaque ressource.
- **OF-35** (ReCyF) : revue des comptes et des droits au moins une fois par an.
- **OF-30 et OF-31** (ANSSI) : secrets distincts par environnement, connus seulement des utilisateurs autorisés.

**Pratiques courantes** (non vérifiées ici) :
- Un compte par processus et par usage, jamais de compte partagé.
- Des identifiants de courte durée, distribués par un coffre et renouvelés régulièrement.
- Pour les pipelines, une identité de charge (OIDC) émise par la CI, sans secret stocké. Son support par Nexus Pro reste à vérifier.

**Ce que Nexus propose** :
- Des jetons d'utilisateur, dont la durée est configurable depuis la 3.69.0.
- Des privilèges et des rôles par dépôt et par sélecteur de contenu.
- Des sources d'identité externes : LDAP et SAML.

### A.2 Alternatives

| | A. Comptes locaux par usage | B. Comptes dans l'annuaire, par groupes | C. Identité de charge (OIDC depuis la CI) |
|---|---|---|---|
| **Principe** | Un compte local Nexus par service et par usage, avec un jeton à durée limitée | Comptes de service dans l'annuaire, droits portés par des groupes | La CI obtient un jeton à la volée, sans secret stocké |
| **Avantages** | Disponible aujourd'hui ; simple à auditer dans Nexus | Centralisé ; revue faite une seule fois, dans l'annuaire | Aucun secret à stocker ni à renouveler ; durée très courte |
| **Inconvénients** | Rotation à organiser à la main ; comptes à maintenir | Dépend de l'annuaire ; le compte de service reste un secret | Support Nexus Pro à vérifier ; dépendance à la CI |
| **Adapté si** | On veut une solution utilisable tout de suite | L'entreprise a déjà un annuaire pour les comptes techniques | Le support OIDC est confirmé sur notre version |

## Annexe B. Exigences vérifiées

Textes vérifiés dans les PDF officiels (ANSSI DevSecOps v1.0, ReCyF v2.5). Les fiches détaillées sont dans `EXIGENCES/`.

| Exigence | Source | Texte (extrait vérifié) | Application aux comptes de service |
|---|---|---|---|
| **OF-10** | ANSSI, DevSecOps v1.0, § « Gérer les secrets » et « Considérer que les actions… » | « appliquer le principe de moindre privilège, de générer à la demande les jetons (tokens), et de journaliser et superviser la CI/CD » | Jetons générés à la demande, droits minimaux pour les pipelines |
| **10.A.1** | ReCyF v2.5, mesure 10.A.1-EI/EE | « …disposent de comptes individuels. Les utilisateurs peuvent, le cas échéant, disposer de plusieurs comptes individuels. » | Un compte par service et par usage, jamais de compte partagé |
| **10.A.2** | ReCyF v2.5, mesure 10.A.2-EI/EE | « L'emploi d'un compte individuel du système d'information est réservé à l'utilisateur ou au processus automatique auquel ce compte a été attribué. » | Chaque compte `svc-…` n'est utilisé que par son pipeline |
| **10.A.3** | ReCyF v2.5, mesure 10.A.3 | Si des comptes individuels ne peuvent être créés pour les processus automatiques, l'entité met en place des mesures de réduction du risque | Non applicable si chaque processus a son compte |
| **10.C.3** | ReCyF v2.5, mesure 10.C.3-EI/EE | « …n'attribue les droits d'accès qu'aux seuls utilisateurs et processus automatiques justifiant d'un besoin au regard de leurs missions. » | Droits par rôle, un sélecteur par application |
| **10.C.4** | ReCyF v2.5, mesure 10.C.4-EI/EE | « …une revue des droits d'accès… au moins annuellement… » | Revue annuelle des comptes et des droits |
| **10.A.6** | ReCyF v2.5, mesure 10.A.6-EI/EE | Revue des comptes ; elle vérifie le respect des mesures relatives à l'identification et corrige les anomalies | Revue du registre des comptes |
| **10.B.4** | ReCyF v2.5, mesure 10.B.4-EI/EE | Le secret d'un compte n'est connu que des utilisateurs autorisés à l'utiliser (exemple : mots de passe des comptes partagés) | Jetons dans le coffre, accès restreint |

**Remarque** : OF-33, OF-34 et OF-35 du référentiel de factice reprennent ces mesures (10.A.1/10.A.2, 10.C.2/10.C.3, 10.A.6/10.C.4). Les textes ci-dessus sont ceux du PDF ReCyF.
