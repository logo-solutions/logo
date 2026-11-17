# Rôle `nexus_access`

Crée les accès de Nexus pour les comptes de service, selon ADR-001 et ADR-002. Il s'inspire de la logique du rôle `roles/nexus` du sous-projet factice (`tasks/security.yml`), mais s'en écarte sur les points suivants.

## Ce que fait le rôle

1. **Sélecteurs** : un par application et par format (`sel-<application>-<format>`).
2. **Privilèges** : un par sélecteur et par dépôt concerné par l'usage.
3. **Rôles** : un par service et par usage (`svc-<service>-<usage>`).
4. **Comptes** : un par service et par usage, avec son rôle.

## Écarts par rapport à factice et à ADR-002

| Point | factice (`roles/nexus`) | Ce rôle | Motif |
|---|---|---|---|
| Rôles | Un par application (`<id>-editeur`, `<id>-lecteur`) | Un par service et par usage | ADR-002, section 2 |
| Sélecteurs | Un par application et par format | Identique | Le préfixe change selon le format (npm `@app`, pypi `packages/`, docker `v2/app/`) ; **écart à ADR-002** (qui dit un par application), à valider |
| Usages | `editeur` et `lecteur` | `publication`, `promotion`, `deploiement`, `lecture` | ADR-002, section 2 |
| Mot de passe | Généré et conservé dans un dossier local | Identique au départ ; jeton à la demande ensuite | OF-10 : jetons à la demande |

## Ce qui reste à faire pour que le rôle fonctionne

- `nexus_applications_flat`, `nexus_privileges_plan`, `nexus_roles_plan` et `nexus_accounts_plan` ne sont pas encore construites à partir de `nexus_services`. Elles devront être calculées dans une tâche de préparation (non écrite ici).
- Les listes des sélecteurs, privilèges, rôles et comptes existants (`nexus_known_*`) sont à lire depuis l'API avant les tâches, comme dans factice.
- Les expressions de sélecteur par format (`defaults/main.yml`) sont à **vérifier sur l'instance** : en particulier les chemins Docker (`v2/`) et pypi (`packages/`).
- L'endpoint de génération de jeton utilisateur n'est pas encore écrit : il faudra le vérifier dans l'interface de l'API.
- Le compte `nexus_admin_username` et son mot de passe viennent du coffre, jamais du dépôt.

## Ne pas oublier

- Le rôle n'écrit rien dans la release sauf pour `promotion` (ADR-002, section 2).
- Aucun compte n'a le droit de supprimer une version publiée (ADR-002, section 7).
- Le rôle est idempotent : un second passage ne doit rien modifier.
