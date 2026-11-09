# Exigences référencées par les ADR Nexus

Une fiche par exigence citée dans `Nexus/ADR-001` et `Nexus/ADR-002`. Chaque fiche donne :
- la source, et son statut de vérification ;
- le texte original, quand il a été vérifié ;
- l'interprétation retenue, à valider par Loïc.

Les textes des référentiels internes d'exigences (OF-xx et EX-xx) sont des reformulations. Ils ne remplacent pas le texte source.

| Fiche | Exigence | Type | Vérification du texte |
|---|---|---|---|
| [OF-22](OF-22.md) | Intégrité du code et signature des tags | Recommandation ANSSI | Texte vérifié |
| [OF-24](OF-24.md) | Cartographie pour identifier les ressources vulnérables | ReCyF 5.A.1 | Texte vérifié |
| [OF-26](OF-26.md) | Téléchargement depuis les sources officielles | ReCyF 5.B.8 | Texte vérifié |
| [OF-61](OF-61.md) | Informations de vérification de l'intégrité | NIST SSDF PS.2.1 | Texte vérifié |
| [EX-17](EX-17.md) | Runners dédiés par zone | Interne | Texte interne ; source ReCyF 7.A.4 vérifiée mais hors sujet direct |
| [EX-22](EX-22.md) | Version publiée jamais modifiée | Interne | Texte interne ; sources externes écartées |
| [EX-24](EX-24.md) | SBOM pour chaque artefact | Interne (CSV) | SSDF PS.3.2 vérifié ; ESF non accessible |

Les textes ont été vérifiés le 7 novembre 2026, à partir des PDF officiels accessibles, et pour SLSA à partir de sa page de spécification.
