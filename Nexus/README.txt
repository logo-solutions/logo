# Nexus Repository Pro : dépôts, chemins et droits

Document de référence pour la mise en place de Nexus Repository Pro, pour trois services et 60 applications. Il est destiné à être approuvé par la communauté de pratique (CoP).

## Par où commencer

| Document | Contenu | Pour qui |
|---|---|---|
| [ADR-001](ADR-001-nommage-depots-chemins.md) | Nommage des dépôts, règle de chemin, promotion, routage Docker | CoP : à approuver |
| [ADR-002](ADR-002-roles-droits.md) | Rôles, droits, suppression sur les releases | CoP : questions à trancher |
| [EXEMPLES.md](EXEMPLES.md) | Applications concrètes, par format et par tier | Tous |
| [STORIES.md](STORIES.md) | Stories de mise en œuvre et critères d'acceptation | Équipe de mise en œuvre |
| [EXIGENCES/](EXIGENCES/README.md) | Exigences citées, avec source et texte vérifié | Relecteurs sécurité |

## Décisions à approuver (ADR-001)

- [ ] Dépôts `<service>-<format>-hosted` (candidat) et `<service>-<format>-release` (release)
- [ ] Chemin `<application>/<composant>/<version>`, sans service dans le chemin
- [ ] Exceptions de format : npm, Maven, Docker, pypi
- [ ] Promotion par republication, avec même empreinte
- [ ] Routage Docker par sous-domaine : 13 sous-domaines
- [ ] Mutualisé et proxys : nommage et groupes

## Décisions à prendre (ADR-002)

- [ ] Suppression sur les releases : A (interdite), B (exploitation avec ticket) ou C (rétention)
- [ ] Comptes de service : par usage ou par service
- [ ] Séparation des runners (EX-17)
- [ ] Administrateurs humains
- [ ] Composants partagés et lecture inter-services

## Ce qui n'est pas traité ici

- Haute disponibilité : hors périmètre.
- Signature des tags et provenance (OF-22, SLSA) : ADR dédiée à créer.
- Sauvegarde et restauration : hors périmètre de ce document.

## Conventions

- Les identifiants `OF-xx` et `EX-xx` renvoient aux fiches de `EXIGENCES/`.
- Les noms d'applications et de services dans les exemples sont illustratifs.
