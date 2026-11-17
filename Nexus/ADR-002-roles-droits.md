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
