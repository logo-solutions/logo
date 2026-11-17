# Comptes de service pour l'accès à Nexus

Topo de référence sur les comptes de service. Les décisions sont reprises dans `ADR-002-roles-droits.md`.

## 1. État de l'art

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

## 2. Trois alternatives

| | A. Comptes locaux par usage | B. Comptes dans l'annuaire, par groupes | C. Identité de charge (OIDC depuis la CI) |
|---|---|---|---|
| **Principe** | Un compte local Nexus par service et par usage, avec un jeton à durée limitée | Comptes de service dans l'annuaire, droits portés par des groupes | La CI obtient un jeton à la volée, sans secret stocké |
| **Avantages** | Disponible aujourd'hui ; simple à auditer dans Nexus | Centralisé ; revue faite une seule fois, dans l'annuaire | Aucun secret à stocker ni à renouveler ; durée très courte |
| **Inconvénients** | Rotation à organiser à la main ; comptes à maintenir | Dépend de l'annuaire ; le compte de service reste un secret | Support Nexus Pro à vérifier ; dépendance à la CI |
| **Adapté si** | On veut une solution utilisable tout de suite | L'entreprise a déjà un annuaire pour les comptes techniques | Le support OIDC est confirmé sur notre version |

## 3. Proposition

**Retenir A pour le lot 1, avec une trajectoire vers C.**

1. **Un compte par service et par usage** : publication, promotion, déploiement, lecture. Trois services, quatre usages, soit au plus 12 comptes, sans compte partagé.
2. **Droits minimaux** : chaque compte n'a que le rôle de son usage, selon ADR-002 (OF-34).
3. **Jeton à durée limitée**, configurée dans Nexus, généré à la demande (OF-10). Il est stocké dans le coffre, jamais dans un dépôt ni dans une variable de CI en clair.
4. **Registre des comptes** : propriétaire, suppléant, usage, date de création, date de dernière revue (ADR-002).
5. **Revue annuelle** des comptes et des droits (OF-35), et rotation des jetons selon la durée retenue.
6. **Vérification de C** : confirmer le support OIDC sur la version Pro. Si c'est le cas, migrer les pipelines de publication et de déploiement, et supprimer les jetons correspondants.

## 4. Décisions prises

- **Propriétaire** : option 1, une personne nommée par compte, avec un suppléant.
- **Suppression sur les releases** : interdite à tous les comptes.
- **Droits par application** : un sélecteur de contenu par application, regroupé dans le rôle du service.
- **Runners** : trois runners distincts (publication, promotion, déploiement), avec des tags distincts.

## 5. Points à trancher

- La durée de vie des jetons (90 jours proposés).
- Le nom des propriétaires et de leurs suppléants.
- Le coffre qui distribuera les jetons aux pipelines.
- Le support OIDC de Nexus Pro.

## 6. Exigences

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
