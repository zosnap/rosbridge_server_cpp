# `p_rosbridge_server_cpp`

[![Build Status](https://jenkins.robopec.com/buildStatus/icon?job=p_rosbridge_server_cpp%2Fmaster)](https://jenkins.robopec.com/view/LIBS%20COMMON/job/p_rosbridge_server_cpp/job/master/)
[![Coverage Status](https://jenkins.robopec.com/buildStatus/icon?job=p_rosbridge_server_cpp%2Fmaster&config=coverage)](https://jenkins.robopec.com/view/LIBS%20COMMON/job/p_rosbridge_server_cpp/job/master/LCOV_20Report/)

Implémentation C++ d'un serveur Websocket du [protocole Rosbridge](https://github.com/RobotWebTools/rosbridge_suite/blob/develop/ROSBRIDGE_PROTOCOL.md).

Il utilise Qt pour l'implémentation Websocket et donc dépend de `libqt5websockets5-dev`.

Les "compressions" **JSON**, **CBOR** et **CBOR-RAW** sont supportés mais pas la compression **PNG**.

### Authentification

Le serveur gérera les messages de requête auth comme spécifié dans le [protocole Rosbridge](https://github.com/RobotWebTools/rosbridge_suite/blob/ros1/ROSBRIDGE_PROTOCOL.md#33-authentication-message). Une instance du nœud de service rosauth doit être en cours d'exécution pour traiter effectivement les requêtes d'authentification.

Si le paramètre require_auth est défini sur true, le serveur ne traitera aucun message (y compris les demandes d'abonnement) tant qu'une authentification réussie n'aura pas été effectuée, et il fermera les sockets qui ne se sont pas authentifiés après un délai d'expiration de 15 secondes.

### Pas implémenté par rapport à la version Python

- compression PNG
- paquets fragmentés
- service server
- Autres transports que Websocket (TCP, UDP)
- Mode BSON only
- Regex pour n'autoriser que certains topics et services

### Différences avec la version Python

En plus des fonctionnalités non implémentées, quelques choix de design diffèrent :

- Le paramètre `type` de la requête `call_service` est obligatoire. La version Python l'ignore et recherche le type avec l'API rosmaster.
- Un watchdog kill le noeud quand la boucle ROS est bloquée pendant plus de 5s

## Détails d'implémentation

### Données binaires

Les tableaux de `int8[]` ou `uint8[]` sont encodés en string codée en base64.

Par exemple le message `sensors_msgs/CompressedImage` suivant :

```python
m.header...
m.format = "jpeg"
m.data = [0,1,2,3,4,5,6,7,8,9]
```

Sera encodé en :

```json
{
    "header": {"seq":123,"stamp":{"secs":123,"nsecs":456},"frame_id":"frame_id"},"format":"jpeg",
    "data":"AAECAwQFBgcICQ=="
}
```

Dans l'autre sens, à partir du JSON, les deux formats sont supportés. Tableau de int ou string en base64.

## Dépendances

- `libqt5websockets5-dev`
- `roscpp`
- `librosqt`
- `ros_babel_fish`
- `rosbridge_cpp_msgs`
- `std_msgs`

## Paramètres optionels

- `port` (*int*) : Port de la websocket. La valeur 0 conduit à ce que l'OS donne un port (défaut = `9090`)
- `service_timeout` (*double*) : Durée du timeout pour les appels de services en secondes (défaut = `5.0`)
- `pong_timeout_s` (*double*) : Durée de timeout, en secondes, avant que l'on déconnecte le client (défaut = `30.0`)
- `watchdog_enabled` (*bool*) : Activation du watchdog (défault = `true`)
- `watchdog_timeout` (*double*) : Timeout du watchdog (défault = `5.0`)
- `require_auth` (*bool*) : Requiert l'authentification (défaut = `false`)
- `require_ssl` (*bool*) : Démarrer le serveur Websocket en mode sécurisé (requiert `ssl_cert_file` et `ssl_key_file` à être donnés) (défaut = `false`)
- `ssl_cert_file` (*string*) : Chemin vers le fichier de certificat SSL (défaut = `""`)
- `ssl_key_file` (*string*) : Chemin vers le fichier de clé SSL (défaut = `""`)

## Paramètres set par le nœud

Le nœud set lui-même le paramètre `actual_port`, utile dans le cas où `port` est mis à `0`.

## Topics publiés

Topics hérités du nœud Python :

- `client_count` (*std_msgs/Int32*) : Nombre de clients connectés, latché et mis à jour à chaque changement du nombre de clients.
- `connected_clients` (rosbridge_msgs/ConnectedClients**) : Information sur les clients actuellement connectés, latché et mis à jour à chaque changement du nombre de clients.

## Tests

Pour tester le fonctionnement avec la boucle d'événements Qt, Qt Test est utilisé.
Mais catkin ne sait utiliser que `gtest` avec `catkin_add_gtest` et `add_rostest_gtest` et donne donc des arguments spécifiques à googletest pour informer du nom du fichier xml de sortie.
Nous modifions donc ces arguments pour les convertir en arguments compris par QTest.
Mais, dans le fichier XML généré par QTest, les `testcase` n'ont pas de champ `time` et rosunit (< 1.15.8) gère mal ce cas, ce qui le fait planter et donc échouer les tests.

En attendant d'utiliser une version corrigée de `rosunit`, il est possible de récupérer la dernière version pour utilisation dans Jenkins :

```groovy
// Install latest version of rosunit to handle p_rosbridge_server_cpp Qt tests
dir('tmp')
{
    sh 'git clone --depth=1 https://github.com/ros/ros'
    sh 'cp -r ros/tools/rosunit ../workspace/src'
}
```
