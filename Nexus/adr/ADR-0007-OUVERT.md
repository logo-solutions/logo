# ADR-0007 (ouvert) : rôles, droits et suppression sur les releases

- **Statut** : ouvert. À reprendre quand les réponses ci-dessous seront disponibles.
- **Dépend de** : ADR-0006 (noms de dépôts, préfixes d'application, groupes).
- **Exigences à intégrer** : EX-17, EX-22, OF-10, OF-12, OF-13, OF-33, OF-34, OF-35 (voir `Nexus/exigences/` et `pilotage/recyf-exigences-officielles.csv` de factice).

## Ce qui est déjà acquis (ADR-0006)

- Dépôts `<service>-<format>-hosted` (candidat) et `<service>-<format>-release`, pour `oig`, `oigs` et `oigl`.
- Dépôts `mutualise-<format>-hosted/release`, proxys `proxy-<format>-<source>`.
- Groupes `<service>-<format>-all` et `mutualise-<format>-all`. Le candidat n'entre jamais dans un groupe.
- Chemin `<application>/<composant>/<version>`, avec les exceptions de format.
- Promotion par republication. Déploiement par empreinte, jamais par tag.

## Point 1 : qui peut supprimer une version en `release` ? (bloquant)

Options proposées précédemment :

| Option | Principe | Avantages | Inconvénients |
|---|---|---|---|
| A. Suppression interdite à tous | Droit retiré ; exception seulement par procédure écrite à deux validations | Immutabilité réelle, traçable | Lourd si une publication erronée doit être corrigée |
| B. Suppression réservée à l'exploitation | Rôle `exploitation`, avec ticket et journal | Corrections possibles, traçables | Dépend de la discipline du rôle |
| C. Rétention automatique | Anciennes versions supprimées après N mois | Stockage maîtrisé | Contraire à l'immutabilité sur la durée |

**Réponse attendue** : A, B ou C, et si A ou B, qui valide l'exception.

## Point 2 : comptes de service par usage

À confirmer pour chaque pipeline :
- compte de **publication candidat** (build) : droit d'écriture sur `<service>-<format>-hosted` seulement ;
- compte de **promotion** : lecture du candidat, écriture sur la release, aucun droit sur le candidat en écriture ;
- compte de **déploiement** : lecture de la release seulement ;
- compte **exploitation** : administration technique, sans publication (et suppression, selon le point 1).

**Question** : les trois services ont-ils un compte de publication et un compte de déploiement distincts, ou un compte par service ?

## Point 3 : séparation des runners (EX-17)

EX-17 demande des runners distincts par zone de sensibilité. Mon interprétation : le runner de build (candidat) et le runner de déploiement (release) sont distincts, avec des tags distincts. Ce n'est qu'une extrapolation de ReCyF 7.A.4.

**Question** : confirmes-tu cette interprétation, et faut-il un runner par service, ou un runner par zone (build, déploiement) ?

## Point 4 : administrateurs humains

- Qui détient le rôle `exploitation` ? Combien de personnes ?
- Faut-il un rôle `ssi-controle` (lecture, gestion des quarantaines et exceptions) ?
- Les comptes humains sont-ils nominatifs (OF-33) ?

## Point 5 : droits par application

Les droits se règlent par préfixe d'application, avec un sélecteur de contenu par application. Pas d'expression régulière nécessaire.

**Question** : une application peut-elle publier sous plusieurs préfixes (ex. un composant partagé entre deux applications) ? Si oui, comment le modéliser ?

## Point 6 : lecture inter-applications

Acquis : la lecture est libre au sein d'un service. Reste à confirmer : un service peut-il lire les releases d'un autre service, ou le mutualisé est-il lisible par tous les services ?

## Ce qu'il faut pour reprendre

1. Choix du point 1 (A, B ou C) et validateurs de l'exception.
2. Confirmation des comptes du point 2.
3. Confirmation de l'interprétation EX-17 et du découpage des runners (point 3).
4. Réponses sur les administrateurs (point 4).
5. Cas des composants partagés (point 5) et lecture inter-services (point 6).

Une fois ces réponses reçues, l'ADR-0007 est rédigée dans ce même dossier, puis la spec §5 est réécrite.
