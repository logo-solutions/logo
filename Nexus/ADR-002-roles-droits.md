# ADR-002 : rôles, droits et suppression sur les releases

- **Statut** : ouvert. Décisions à prendre avant rédaction.
- **Dépend de** : ADR-001 (noms de dépôts, préfixes d'application, groupes)
- **Exigences à intégrer** : EX-17, EX-22, OF-10, OF-12, OF-13, OF-33, OF-34, OF-35 (voir `EXIGENCES/`)

---

## En bref (pour la CoP)

Les rôles et droits ne peuvent pas être figés tant que quatre questions ne sont pas tranchées. Les points 1 et 3 bloquent le reste. Chaque point ci-dessous présente les options et une question à approuver.

---

## Ce qui est acquis (ADR-001)

- Dépôts `<service>-<format>-hosted` (candidat) et `<service>-<format>-release`, pour `oig`, `oigs` et `oigl`.
- Dépôts `mutualise-<format>-hosted/release`, proxys `proxy-<format>-<source>`.
- Groupes `<service>-<format>-all` et `mutualise-<format>-all`. Le candidat n'entre jamais dans un groupe.
- Chemin `<application>/<composant>/<version>`, avec les exceptions de format.
- Promotion par republication. Déploiement par empreinte.

---

## Point 1 : suppression d'une version en `release` (bloquant)

| Option | Principe | Avantages | Inconvénients |
|---|---|---|---|
| **A. Suppression interdite à tous** | Le droit est retiré. Exception seulement par procédure écrite, avec deux validations. | Immutabilité réelle et traçable | Lourd si une publication erronée doit être corrigée |
| **B. Suppression réservée à l'exploitation** | Rôle `exploitation`, avec ticket et journal. | Corrections possibles et traçables | Dépend de la discipline du rôle |
| **C. Rétention automatique** | Les anciennes versions sont supprimées après N mois. | Stockage maîtrisé | Contraire à l'immutabilité sur la durée |

**Question à approuver** : option A, B ou C ? Si A ou B, qui valide l'exception ?

---

## Point 2 : comptes de service

Un compte par usage, pour chaque pipeline :

| Compte | Droits |
|---|---|
| Publication candidat (build) | Écriture sur `<service>-<format>-hosted` seulement |
| Promotion | Lecture du candidat, écriture sur la release, aucune écriture sur le candidat |
| Déploiement | Lecture de la release seulement |
| Exploitation | Administration technique, sans publication ; suppression selon le point 1 |

**Question à approuver** : un compte de publication et un compte de déploiement par service, ou un compte par service pour l'ensemble des usages ?

---

## Point 3 : séparation des runners (EX-17, bloquant)

EX-17 demande des runners distincts par zone de sensibilité. Interprétation proposée : le runner de build (candidat) et le runner de déploiement (release) sont distincts, avec des tags distincts. La séparation des dépôts ne suffit pas.

Source : ReCyF 7.A.4 traite du cloisonnement des sous-systèmes, pas des runners. L'application aux runners est donc une extension, à valider (voir `EXIGENCES/EX-17.md`).

**Question à approuver** : cette interprétation est-elle acceptée, et le découpage se fait-il par service ou par zone (build, déploiement) ?

---

## Point 4 : administrateurs humains

- Qui détient le rôle `exploitation` ? Combien de personnes ?
- Faut-il un rôle `ssi-controle` (lecture, gestion des quarantaines et exceptions) ?
- Les comptes humains sont-ils nominatifs (OF-33) ?

**Question à approuver** : liste nominative des détenteurs du rôle `exploitation`, et création ou non du rôle `ssi-controle`.

---

## Point 5 : droits par application

Les droits se règlent par préfixe d'application, avec un sélecteur de contenu par application. Les expressions régulières ne sont pas nécessaires.

**Question à approuver** : une application peut-elle publier sous plusieurs préfixes, par exemple pour un composant partagé entre deux applications ? Si oui, comment le modéliser ?

---

## Point 6 : lecture inter-services

Acquis : la lecture est libre au sein d'un service.

**Question à approuver** : un service peut-il lire les releases d'un autre service ? Le mutualisé est-il lisible par tous les services ?

---

## Ce qu'il faut pour rédiger l'ADR

1. Choix du point 1 (A, B ou C) et validateurs de l'exception.
2. Confirmation des comptes du point 2.
3. Validation de l'interprétation EX-17 et du découpage des runners (point 3).
4. Réponses sur les administrateurs (point 4).
5. Cas des composants partagés et lecture inter-services (points 5 et 6).

Une fois ces réponses reçues, l'ADR-002 est rédigée, puis la section RBAC de la spec est reprise.
