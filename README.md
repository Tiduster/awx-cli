# Prérequis
Ces scripts fonctionnent sous **Python3** et nécessitent la librairie **requests** > 2.4, **json** et **yaml**

# TOWER API
Le script a besoin d'un fichier **secret.py**, à placer à côté de **tower.py**.

Voici la syntaxe attendue :
```
# secret.py:
username = 'cpatry'
secret = 'p4ssw0rd$'
```

### Usage du script :
```
./tower.py
usage: tower.py [-h] [--verbosity {INFO,ERROR,DEBUG}]
    tower.py addHost fqdn inventory_name
    tower.py massImportHost txtfile (file syntax: fqdn;inventory_name)
    tower.py massImportInventory txtfile (file syntax: inventory_name;organisation)
    tower.py deleteHost fqdn
    tower.py massDeleteHost txtfile (file syntax: fqdn)
    tower.py massChangeHostStatus txtfile (file syntax: fqdn;enabled[true/false])
    tower.py addGroup [--force-create] group_name inventory_name
    tower.py deleteGroup group_name inventory_name
    tower.py addInventory [--org [ORGANIZATION_NAME]] [--force-create] inventory_name
    tower.py deleteInventory inventory_name
    tower.py deleteJobTemplate job_template_name
    tower.py associate fqdn group_name inventory_name
    tower.py associateVariable [type {hosts,groups}] name key value inventory_name
    tower.py associateChildren group_parent_name group_child_name inventory_name
    tower.py massAssociate txtfile (file syntax: fqdn;group_name;inventory_name)
    tower.py disassociate fqdn group_name inventory_name
    tower.py hostGroups fqdn
    tower.py groupMembers [--export] group_name inventory_name
    tower.py groupListMembers [--inverse] inventory_name group_list [group_list ...]
    tower.py groupVars group_name inventory_name
    tower.py hostVars [--nested] fqdn
    tower.py lastExecutionStatus id (of the JobTemplate)
    tower.py lastExecutionChange id (of the JobTemplate)
    tower.py getAllGroupVars
    tower.py getAllHostVars
    tower.py getLonelyHosts
    tower.py getLonelyGroups
    tower.py getTemplatesWithSurvey
    tower.py getTemplatesNotUsingDefault
    tower.py getProjectsWithOldBranch
    tower.py displayJobTemplatesCredentials
    tower.py launchJob [--remote_username REMOTE_USERNAME]
                       [--remote_password REMOTE_PASSWORD]
                       [--tags TAGS] [--inventory INVENTORY]
                       [--limit LIMIT] [--si_version SI_VERSION]
                       [--job_type JOB_TYPE] [--disable_cooldown]
                       [--return_id]
                       jsonfile
    tower.py stopJob job_id
    tower.py importAnsibleInventory [--export]
                                    [--org [ORGANIZATION_NAME]]
                                    [--groupvars [GROUP_VAR_DIRECTORY_PATH]]
                                    [--hostvars [HOST_VAR_DIRECTORY_PATH]]
                                    file inventory
    tower.py exportAnsibleInventory --bash jsonfile inventory_name
    tower.py importJobTemplates jsonfile
    tower.py exportAllJobTemplates jsonfile
    tower.py importAll --delete jsonfile {credentials,projects}
    tower.py exportAll jsonfile {credentials,projects,disabledHosts}
```

# getProjectsWithOldBranch

```
curl https://gitlab/api/v4/projects/3/repository/branches?per_page=100
cat branches.json | jq -r .[].name > branchList
```

# LAUNCH JOB
La fonctionnalité launchJob permet d'exécuter un job template sur l'interface Ansible Tower.
Elle fonctionne à l'aide de fichiers json, qui sont déposés dans le dossier **yaml_launch**.
J'en ai déjà créé plusieurs. Bien sur le but c'est que chacun fasse les siens et les partage !

Prenons par exemple le fichier **autolib-ehp.yaml** :
```
job_template_id:
  - 607
survey:
  run_default_migrations: 'no'
  report: 'Non'
  validation: 'Non'
  microlise_environment: 'eh1'
```
Il contient l'id du job template **"EHP - Autolib"**, ainsi que les réponses au survey (qui sont obligatoires).

Pour exécuter le job, il suffit de lancer la commande suivante (sans le .yaml) :
```
tower.py launchJob autolib-ehp
```

Vous pouvez exécuter simultanément plusieurs jobs en les listant comme ceci :
```
job_template_id:
  - 607
  - 431
  - 720
  ```
Voir par exemple le fichier d'exemple **dirtyharry-ehp.yaml**.

Si vous souhaitez modifier la limite d'un job, vous pouvez utiliser le paramètre **--limit**.
Attention, ce paramètre fonctionnera uniquement si le job template demande la limite au moment de l'exécution, autrement vous n'avez pas le droit de la modifier.

La plupart des paramètres usuels peuvent être modifiés, à partir du moment ou le job template le permet. C’est par exemple le cas pour :

- si_version
- tags
- inventory
- job_type

Vous pouvez indiquer ces paramètres en les ajoutant à la variable **survey**, comme dans le fichier d’exemple **autolib-ehp.yaml**. Vous pouvez aussi les mettre directement dans la ligne de commande, comme dans l’exemple suivant :

```
tower.py launchJob psql-database --limit server01 --inventory 1
```

Enfin, par défaut, tower.py ne lancera pas plus de 3 jobs simultanéments. Il effectuera une courte pause tous les 3 jos templates. Si vous souhaitez désactiver ce comportement, utiliser l’argument **--disable_cooldown**.

# TOWER MIGRATION

Pour migrer les données vers AWX, procédez dans l'ordre suivant :

- Création manuelle des organisations
- Import automatique des inventaires
- Créer le type credential PHPIpam
- Import automatique des credentials après sed sur l'id phpipam credential_type + id organization:
    -> Système :

```jq '. | map(select(.id == (14, 15, 16, 22, 26, 28, 35, 37, 41, 55, 65, 66, 69, 70, 72, 79, 81, 94, 95, 103, 109, 115, 116, 121, 122, 124, 125, 126, 127, 134, 135)))' credentials > credentials.json```

    -> Chaque user recréera ses credentials
- Import automatique des projets après sed sur l'id de l'organisation + l'id des credentials
- Correction des credentials SCM + ajout system creds 
- Import automatique des hosts et groupes via inventory script :

```
#!/bin/bash
if [ "$1" == "--list" ] ; then
cat << "EOF"
{
    JSONFILE EXPORTED
}
EOF
elif [ "$1" == "--host" ]; then
  echo '{"_meta": {"hostvars": {}}}'
else
  echo "{ }"
fi
```

```./tower.py exportAnsibleInventory inventories/EHP-AWS "EHP - AWS" --bash```

- Import automatique des Job Templates après sed des ids d'inventaire, de credentials et de projects
- Suppression des source_inventory, dans la base, puis via l'interface :

```
UPDATE main_group SET has_inventory_sources = 'f' where has_inventory_sources = 't';
UPDATE main_host SET has_inventory_sources = 'f' where has_inventory_sources = 't';

TRUNCATE TABLE main_group_inventory_sources;
TRUNCATE TABLE main_host_inventory_sources;

```

- Remise en place des lonelyGroups
- Désactivation des serveurs anciennement désactivés

```./tower.py exportAll disabledHosts disabledHosts && jq -r .[].name disabledHosts > disabledHosts.list```

# IMPORT INVENTORY
Variable syntax : key: value (ex: key1: value, key2: value, etc...)
