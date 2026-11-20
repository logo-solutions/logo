# Nommage des dépôts Nexus : décisions, raisons et alternatives

> **En bref.** Rien dans les contraintes n'exige de distinguer les trois services dans Nexus. La séparation des droits n'est pas demandée, l'administration est la même pour les trois, et le nom d'application est déjà unique grâce à la CMDB. Mettre le service dans le nom des dépôts n'apporterait donc que de la lisibilité, au prix d'une migration à chaque réorganisation. Un périmètre unique supprime ce coût sans en créer d'autre : une nouvelle application ne demande aucune configuration, une réorganisation non plus. La direction, elle, reste dans le nom des dépôts : l'instance Nexus est partagée par toute la société, et c'est entre directions que les comptes, les droits et les règles changent.

*Document destiné à la communauté de pratique. Il explique pourquoi les dépôts Nexus sont découpés par direction, et non par service ni pour toute la société, puis détaille l'organisation retenue, ses limites et les alternatives.*

## 1. Pourquoi un périmètre par direction, et pas par service

L'instance Nexus est partagée par toute la société. Les dépôts sont découpés au niveau où la configuration Nexus change, et pas plus finement. Ce principe tranche dans les deux sens. Entre les trois services, rien ne change : le service est retiré. Entre directions, les comptes, les droits, les règles de nettoyage et les responsables changent : la direction est gardée.

### Le raisonnement

1. **Aucune contrainte ne sépare les services.** La séparation des droits n'est pas exigée et l'administration Nexus est identique pour les trois services.
2. **Le service n'est pas nécessaire pour identifier une application.** Le nom d'application est unique sur les trois services, et c'est la CMDB qui le garantit.
3. **Le service est l'information la plus instable.** Il change avec les réorganisations, et Nexus ne sait pas renommer un dépôt. Avec le service dans le nom, chaque réorganisation impose de créer de nouveaux dépôts, d'y copier les artefacts et de changer les adresses chez tous les clients.
4. **L'avantage des trois services existe aussi avec un seul périmètre.** Avec le service dans le nom des dépôts, une nouvelle application ne demande aucune configuration. Cet avantage ne compte que face à des sélecteurs de contenu, qu'il faut mettre à jour à chaque création pour reconstruire une vue par service. Si personne n'a besoin de cette vue, un périmètre unique offre le même avantage, sans le coût des réorganisations.
5. **Sans la direction, les dépôts deviennent ceux de toute la société.** Les noms de dépôts sont uniques sur l'instance. Un dépôt `npm-release` passe pour le dépôt commun de la société : d'autres équipes y publieraient, et la première direction qui voudrait ses propres dépôts devrait adopter un préfixe.
6. **Partager les dépôts avec toute la société casse le modèle de droits.** Dans Nexus, un droit porte sur un dépôt entier. Un compte de publication de la direction pourrait publier pour n'importe quelle application de la société. Pour le limiter aux 60 applications, il faudrait des sélecteurs de contenu sur chaque compte, alimentés par la CMDB. Les comptes des autres directions accéderaient aux artefacts de la direction, sauf si le RA Nexus refaisait leurs droits : la direction dépendrait de modifications hors de son périmètre. La lecture s'étendrait aussi à toute la société, ce que le besoin d'en connaître peut interdire.

### Comparaison des trois découpages

| Critère | Par service | Par direction (retenu) | Toute la société |
|---|---|---|---|
| Nouvelle application | Rien à faire | Rien à faire | Mise à jour des sélecteurs des comptes de la direction |
| Application qui change de service | Copie de son historique vers d'autres dépôts | Rien à faire | Rien à faire |
| Fusion, scission ou renommage d'un service | Nouveaux dépôts, copie des artefacts, nouvelles adresses chez les clients | Rien à faire | Rien à faire |
| Fusion ou scission de la direction | Sans effet | Migration des dépôts | Sans effet |
| Droits des comptes de service | Simples, par dépôt | Simples, par dépôt | Sélecteurs de contenu sur chaque compte, y compris ceux des autres directions |
| Lecture | Par service | Par direction | Toute la société |
| Règles de nettoyage et de conservation | Par service | Par direction | Communes à la société |
| Applications touchées par une erreur de configuration sur un dépôt | 20 | 60 | Toute la société |
| Sous-domaines Docker | 8 | 4 | Partagés avec la société |
| Service lisible dans le nom du dépôt | Oui | Non : il est dans la CMDB | Non |

### Ce que les trois services apportent et pourquoi cela ne suffit pas

- **La lisibilité.** Le gain est réel, mais le nom du dépôt affiche l'information qui change le plus souvent. La CMDB la donne déjà.
- **Un périmètre d'incident de 20 applications au lieu de 60.** L'administration est commune : les erreurs les plus graves (rôles, stockage, règles de nettoyage) touchent déjà les trois services. Avec 10 builds par jour, le gain est faible.
- **Les groupes de lecture par service.** Comme la séparation des droits n'est pas exigée, c'est un confort d'usage. Un groupe global sans les candidats suffit.
- **Les comptes de service par service.** Ils n'ont pas d'autre justification que la cohérence de nommage.

### Choix du préfixe

Le préfixe doit être un code de périmètre stable, et non le nom affiché de la direction. Un renommage de la direction ne coûte alors rien. Une fusion ou une scission impose toujours une migration.

## 2. Organisation retenue

- **Nom du dépôt** : `<direction>-<format>-<cycle>`, par exemple `<direction>-npm-release` ou `<direction>-docker-candidat`. Le préfixe est le code de la direction qui regroupe les trois services (voir la section 1).
- **Chemin dans le dépôt** : il commence par le nom de l'application, par exemple `dbx/…`. Ce nom vient de la CMDB, qui garantit son unicité. L'identifiant CMDB à 10 caractères n'est pas utilisé, car il est illisible dans l'arborescence.
- **Ressources partagées** : dépôts séparés `mutualise-<format>-<cycle>` pour les images de base et les bibliothèques communes.
- **Groupes** : `<direction>-<format>-all` expose les releases, sans les candidats.

```text
<direction>-npm-candidat   dépôt hébergé : dbx/…, <autre-application>/…
<direction>-npm-release    dépôt hébergé, immuable
<direction>-npm-all        groupe : releases, sans les candidats
mutualise-docker-release   images de base communes
```

### Pourquoi le chemin commence par l'application

1. **Les coordonnées d'un artefact ne dépendent pas de l'organisation.** Le nom d'une image (`dbx/api:1.4.2`) ou le scope d'un paquet npm (`@dbx/…`) reste le même si l'application change de service. Les clients n'ont aucune référence à modifier.
2. **Le lien avec la CMDB est direct.** Le premier niveau du chemin est le nom d'application de la CMDB. Un artefact, son SBOM et ses vulnérabilités se rattachent à l'application sans table de correspondance.
3. **Un préfixe suffit pour filtrer.** Les sélecteurs de contenu et les scripts d'inventaire travaillent sur un simple préfixe (`^/dbx/`, ou `^/v2/dbx/` en Docker). C'est ce qui permet d'ajouter plus tard une vue par service sans déplacer aucun artefact.
4. **La convention est la même pour tous les formats.** Elle se traduit par l'espace de noms de l'image en Docker, le scope en npm et le premier dossier en raw. En npm, cela impose aux builds de publier sous le scope de l'application.

### Pourquoi des dépôts séparés pour les ressources partagées

Les images de base et les bibliothèques communes pourraient être un simple chemin `mutualise/` dans les dépôts de la direction. Elles restent dans des dépôts séparés pour deux raisons :

1. **Protéger ce que toutes les applications consomment.** Une image de base corrompue contamine les 60 applications. Avec des dépôts séparés, la publication est réservée à ceux qui produisent ces ressources, par de simples droits sur les dépôts, sans sélecteur de contenu. Aucun pipeline applicatif ne peut y publier. Il ne s'agit pas d'une séparation des droits entre services, mais de la protection de la chaîne d'approvisionnement.
2. **Conserver ces ressources plus longtemps.** Les règles de nettoyage s'appliquent par dépôt. Les images de base peuvent ainsi avoir une durée de conservation différente de celle des livrables applicatifs.

Le coût est de 2 sous-domaines Docker de plus, jusqu'au passage au routage par chemin.

**Ce choix est à faire confirmer par le RA Nexus.**

### Pourquoi un sous-domaine par dépôt Docker

Un client Docker attend le registre à la racine d'un hôte (`/v2/`). Nexus propose trois façons de servir chaque dépôt Docker : un port par dépôt, un sous-domaine par dépôt, ou un routage par chemin dans les versions récentes.

1. **Un seul port à ouvrir.** Tous les dépôts passent par le reverse proxy en 443, sans règle de pare-feu par dépôt.
2. **Des références d'images lisibles.** `<direction>-docker-release.<domaine>/dbx/api:1.4.2` indique le dépôt, l'application et la version.
3. **Un coût marginal quasi nul.** Avec un certificat et un enregistrement DNS wildcard, un nouveau dépôt Docker ne demande aucune action sur le DNS ni sur les certificats.

A priori, la version de Nexus actuellement en place ne propose pas le routage par chemin. Dès qu'il sera disponible, c'est l'option qui sera retenue : elle supprime ces sous-domaines. Ce passage changera les références d'images côté clients, puisque le nom du dépôt passera de l'hôte au chemin (`<domaine>/<direction>-docker-release/dbx/api:1.4.2`). Il faudra donc prévoir une période de transition. Si la montée de version de Nexus est proche, il est préférable d'attendre avant de passer les 60 applications en Docker, pour ne pas faire cette migration deux fois.

## 3. Contraintes de départ

| Contrainte | Source |
|---|---|
| Trois services (`oig`, `oigs`, `oigl`), environ 20 applications chacun, soit 60 applications | Échanges de cadrage |
| Périmètre limité à ces trois services, sans extension prévue | Commanditaire |
| Administration Nexus identique pour les trois services | Commanditaire |
| Images de base partagées par les trois services | Commanditaire |
| Lecture libre entre les applications d'un même service | Commanditaire |
| Volume faible : 10 builds par jour au total | Commanditaire |
| Nom d'application unique sur les trois services | CMDB |
| Identifiant CMDB à 10 caractères, illisible dans l'arborescence | Échanges de cadrage |
| La direction change moins souvent que ses services | Hypothèse de travail |
| Instance Nexus partagée par toute la société | Échanges de cadrage |
| Aucune convention de nommage imposée par le RA Nexus | Échanges de cadrage |

**Non exigé** : la séparation des droits entre services.

## 4. Limites assumées

- **Le service n'apparaît plus dans Nexus.** Pour savoir à quel service appartient une application, il faut consulter la CMDB.
- **La direction figure dans le nom des dépôts.** Si elle est fusionnée ou scindée, les dépôts doivent être migrés. C'est le risque écarté pour les services, déplacé à un niveau où il est plus rare. Un simple renommage ne coûte rien si le préfixe est un code de périmètre stable.
- **La lecture libre s'étend aux 60 applications.** La contrainte initiale ne la prévoyait qu'au sein d'un service. Ce point doit être validé par le commanditaire.
- **Une erreur de configuration sur un dépôt touche les 60 applications**, au lieu de 20.
- **Une seule règle de nettoyage par dépôt** pour toutes les applications, et pas de suivi du stockage par service.

## 5. Conditions de validité

Ce choix tient tant que les trois conditions suivantes sont vraies :

1. Aucun service n'a besoin d'une durée de conservation différente.
2. Personne n'a besoin de suivre ou de refacturer le stockage par service.
3. Aucun utilisateur n'a besoin d'une vue limitée à un service.

Si l'une d'elles cesse d'être vraie, on ajoute des sélecteurs de contenu (alternative C), sans déplacer aucun artefact. On ne revient aux trois services que si les sélecteurs ne suffisent pas, par exemple si les règles de nettoyage doivent différer.

## 6. Alternatives et impacts

| Alternative | Principe | Gain | Impact |
|---|---|---|---|
| **A. Retenue : un seul périmètre** | `<direction>-npm-release`, chemin `dbx/…` | Aucun coût pour une nouvelle application ni pour une réorganisation des services, 4 sous-domaines Docker | Service invisible dans Nexus, 60 applications par dépôt, migration si la direction change |
| **B. Service dans le nom du dépôt** | `oigs-npm-release`, chemin `dbx/…` | Lisibilité, périmètre d'incident de 20 applications, groupes par service | Migration des dépôts à chaque réorganisation, 8 sous-domaines Docker. Écartée |
| **C. Un seul périmètre, avec sélecteurs par service** | `<direction>-npm-release`, une vue par service générée depuis la CMDB | Vue ou droits par service, sans migration | Un script entre la CMDB et l'API REST de Nexus, à lancer à chaque nouvelle application. Solution de repli |
| **D. Service dans le chemin** | `<direction>-npm-release`, chemin `oigs/dbx/…` | Aucun dépôt à migrer | Le service entre dans les coordonnées des artefacts (noms d'images, scopes npm) : à chaque réorganisation, les artefacts sont déplacés et les clients changent leurs références. Écartée |
| **E. Un dépôt par application** | `dbx-npm-release` | Isolation maximale | 360 à 480 dépôts et autant de sous-domaines Docker. Écartée |

### Fonctionnement des sélecteurs, si le besoin apparaît

1. **Sélecteur de contenu** : une expression qui ne garde que les chemins des applications d'un service.
2. **Privilège** : il associe le sélecteur aux dépôts voulus, en lecture seule.
3. **Rôle** : il regroupe ce privilège et il est attribué au compte concerné.

La liste des applications ne doit jamais être tenue à la main : un script lit la CMDB et réécrit les sélecteurs par l'API REST de Nexus. Si le script n'est pas lancé, une nouvelle application reste invisible : l'erreur se voit tout de suite et elle n'ouvre aucun accès en trop.

**Cas de Docker.** Nexus range les couches des images dans un espace commun (`/v2/-/blobs/`), partagé par toutes les images du dépôt. Le sélecteur doit autoriser `/v2/` pour `docker login` et l'accès aux couches : il ne filtre donc que les manifestes et les tags.

## 7. Décisions

- **Un seul périmètre pour les trois services : retenu.** Les dépôts portent le code de la direction (`<direction>-<format>-<cycle>`), pas celui du service. Le service reste dans la CMDB.
- **Code de la direction : un code de périmètre stable**, et non le nom affiché de la direction.
- **Service dans le nom des dépôts : écarté**, pour les raisons de la section 1.
- **Séparation des droits entre services : non exigée.** Si elle le devient, elle passe par des sélecteurs de contenu.
- **Ressources partagées : dépôts `mutualise-*` séparés**, sous réserve de confirmation par le RA Nexus.
- **Routage Docker : par sous-domaine aujourd'hui, par chemin dès que la version de Nexus le permettra.**
- **Comptes de service par service : abandonnés.** L'ADR-002 est à réviser pour des comptes par usage (publication, lecture).

## 8. Points ouverts

- **Lecture libre sur les 60 applications** : à faire valider par le commanditaire.
- **Convention de nommage sur l'instance** : le RA Nexus n'en impose aucune. Lui proposer `<direction>-<format>-<cycle>` comme convention commune et lui faire réserver le code de la direction. Sinon, une autre équipe peut créer un dépôt du même nom ou reprendre le même préfixe.
- **Dépôts séparés pour les ressources partagées** : à faire confirmer par le RA Nexus, avec leur préfixe (`mutualise-…` ou `<direction>-mutualise-…`).

---

*Version du 20 novembre 2026.*
