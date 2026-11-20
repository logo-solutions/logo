# Pourquoi le service figure dans le nom des dépôts

Document destiné à la communauté de pratique. Il retrace les contraintes qui ont conduit à nommer les dépôts `<service>-<format>-<cycle>`, puis met cette décision à l'épreuve. Les décisions détaillées sont dans `ADR-001-nommage-depots-chemins.md`.

## 1. Contraintes de départ

| Contrainte | Source |
|---|---|
| Trois services (`oig`, `oigs`, `oigl`), environ 20 applications chacun, soit 60 applications | Échanges de cadrage |
| Administration identique sur les trois services | Réponse du commanditaire |
| Images de base partagées par les trois services (`mutualise`) | Réponse du commanditaire |
| Lecture libre entre applications d'un même service | Réponse du commanditaire |
| Volume faible : 10 builds par jour au total | Réponse du commanditaire |
| Chemin commençant par l'application, sans le service | ADR-001, section 3 |
| Routage Docker par sous-domaine, un par dépôt et par groupe | ADR-001, section 5 |
| Identifiant CMDB à 10 caractères, illisible dans l'arborescence | Échanges de cadrage |
| Renommage des dépôts accepté en cas de réorganisation | Décision du commanditaire |

**Non exigé** : la séparation des droits entre services. C'est une préférence, pas une contrainte.

## 2. Solutions écartées avant le service dans le nom

- **Un dépôt par application** : 360 à 480 dépôts hébergés, administration lourde, autant de sous-domaines Docker. Écarté.
- **Un dépôt générique par format, sans service** (ex. `docker-candidat`) : ne permet pas de lire un périmètre sans connaître les 60 applications. Écarté.
- **Clé par identifiant CMDB** : illisible dans l'arborescence. Écarté au profit de la clé `service.app`, figée à vie.

## 3. Pourquoi le service dans le nom

1. **Périmètre d'un incident** : une erreur de configuration touche 20 applications au lieu de 60.
2. **Lecture** : un nom de dépôt porte deux informations (service, format), plus le cycle, sans consulter la CMDB.
3. **Groupes de lecture par service** : `<service>-<format>-all` expose la release d'un service, sans son candidat.

La séparation des droits n'est pas une raison retenue (voir section 1).

## 4. Mise à l'épreuve

**a. Le service change avec l'organisation.** Un service peut être fusionné, scindé ou renommé. Dans ce cas, ses dépôts sont renommés et les clients migrés. Le commanditaire a accepté ce coût : la décision est donc tenable, mais elle n'est pas gratuite.

**b. La redondance.** Le service est dans le nom du dépôt, et l'application dans le chemin. Un lecteur qui connaît l'application doit d'abord connaître son service pour trouver son dépôt.

**c. Le coût DNS en Docker.** Avec service, il y a 8 sous-domaines Docker (6 pour les services, 2 pour le mutualisé). Sans service, il y en aurait 2 à 4.

**d. Le périmètre d'incident est réduit, pas supprimé.** Les trois services partagent la même administration. Une erreur dans cette administration touche les trois services.

## 5. Alternatives

| Alternative | Principe | Gain | Coût |
|---|---|---|---|
| **A. Retenue : service dans le nom** | `oigs-npm-release` | Périmètre, lisibilité, groupes par service | Renommage des dépôts en cas de réorganisation ; redondance avec le chemin |
| **B. Par format et cycle seulement** | `npm-release`, service dans le chemin | Pas de renommage si un service change | Le chemin devient `oigs/dbx/...`, contraire à la règle « application en premier » |
| **C. Par format, sans service, droits par sélecteur** | `npm-release`, sélecteurs par application | Peu de dépôts, peu de sous-domaines | Un seul périmètre pour les 60 applications |

## 6. Décisions

- **Service dans le nom : retenu (alternative A).** Le renommage des dépôts en cas de réorganisation est accepté.
- **Séparation des droits entre services : non exigée.** Les comptes de service restent par service (ADR-002) par cohérence de nommage, sans que ce soit une exigence.

## 7. Points ouverts

- La procédure de renommage : qui la déclenche, et comment les clients sont prévenus.
- La question des comptes par service : les garder tels quels, ou les regrouper si la séparation n'est pas une exigence.
