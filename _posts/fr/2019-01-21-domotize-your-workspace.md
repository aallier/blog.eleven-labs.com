---
layout: post
title: Domotiser son espace de travail
excerpt: L'objectif de cet article est de domotiser simplement et efficacement son espace de travail avec home-assistant.
authors:
    - pouzor
lang: fr
permalink: /fr/domotize-your-workspace/
categories:
    - domotique
    - gitlab
tags:
    - homeassistant
    - gitlab-ci
    - domotique
---

Alors que la domotique “grand public” s’installe de plus en plus dans les foyers, notamment grâce aux nouveaux assistants personnels (Google Home, Alexa) ou encore aux solutions de domotisation “clefs en main”, elle est plutôt absente dans les environnements pro, et particulièrement dans notre cas celui du développement. Pourtant les choses ont pas mal bougés ces dernières dans ce domaine.

Dans cet article, nous allons voir comment construire un dashboard d’équipe, à l’aide de la solution domotique [home-assistant.io](https://www.home-assistant.io) et de quelques objets connectées (Philips Hue, Google Home).

## home-assistant.io

Mais qu’est-ce donc que home assistant ? Home Assistant (hass) est une solution domotique/automatisation open source, privacy-first qui commence à se faire vraiment bien connaître dans le monde de la domotisation. Il est développé en python, basé sur un système événementielle et “packagé” avec l’ensemble des plugins (pas d’installation de plugin à posteriori).

Nous allons tout d’abord installer hass, et pour se faire, plusieurs choix se portent à nous (Docker, hassbian ou hass.io). Dans le cadre de ce tuto, nous allons le faire tourner sous docker mais je vous invite à tester hassbian qui marche très bien sous raspberry.

Cela tient en deux commandes (sous linux) : 

```sh
$ mkdir home-assistant
$ docker run -d --name="home-assistant" -v /PATH_TO_YOUR_FOLDER_HASS:/config -e "TZ=Europe/Paris" -p 8123:8123 homeassistant/home-assistant
```
Toutes les infos sur l’installation sont disponibles [ici](https://www.home-assistant.io/docs/installation/docker/) 


Et voilà, votre home-assistant tourne maintenant en local, et vous pouvez y accéder ici : 
http://localhost:8124/

Il vous suffit maintenant simplement de créer votre login/pass pour accéder à hass.
Nous allons donc voir maintenant comment activer un [component](https://www.home-assistant.io/components/), en commençant avec [sensor.gitlab_ci](https://www.home-assistant.io/components/sensor.gitlab_ci/).

## Components et Sensor Gitlab CI

Tout d’abord, un petit laïus sur comment fonctionne home-assistant.

Si vous regardez dans le répertoire home-assistant que vous avez créé dans la première étape, vous devriez voir ceci : 

```sh
$ ls home-assistant

automations.yaml     customize.yaml       groups.yaml          home-assistant_v2.db secrets.yaml
configuration.yaml   deps                 home-assistant.log   scripts.yaml         tts

``` 

Globalement, toute la configuration se passe dans le fichier `configuration.yaml`, les autres .yaml étant inclus dans celui ci.

Comme vous pouvez le voir, celui-ci est déjà rempli avec un certain nombre d’éléments, éléments qui sont déjà visible sur votre home-assistant (map, plugin jour/nuit et météo).

Pour faire simple, il existe plusieurs types de component installable sur hass, celui qui va nous intéresser ici c’est le type sensor, celui de gitlab-ci :

Dans la partie sensor du fichier configuration.yaml :

```yaml
sensor:
  - platform: gitlab_ci #référence du plugin
    gitlab_id: xxx #ID du projet à monitorer
    token: xxx #token gitlab
    name: Test gitlab projet X # va etre l'id de votre sensor

```


On va maintenant vérifier que le fichier yaml est correcte, puis relancer home-assistant pour prendre la configuration en compte.





Vous devriez maintenant voir apparaître le dernier statut du build du projet.


Et si vous cliquez sur le sensor, vous aurez plus d’infos : 




## Design

Bon c’est bien sympa mais on ne voit pas grand chose et c’est assez minimaliste tout ca. Allons ajouter un coup de template la dessus.
Je vous propose d’utiliser [lovelace](https://www.home-assistant.io/lovelace/), le nouveau système de templating de hass. Pourquoi ? Parce qu’il dispose d’un mode yaml et donc quitte à faire de la configuration, autant aller jusqu’au bout.

On va commencer par activer lovelace, et le setter par défaut.



Apres avoir recharger la page, lorsque vous retournerez sur “Aperçu”, l’url sera maintenant `/lovelace/0` mais le contenue reste pour le moment le même.

Nous allons aussi activer le mode yaml de lovelace.

````yaml
lovelace:
  mode: yaml
``` 

Et créer le fichier de configuration qui va bien dans le répertoire racine.

```sh
$ touch ui-lovelace.yaml
```

Next, nous avons besoin de faire apparaitre les valeurs secondaires du plugin gitlab comme un sensor à part entière, nous allons donc les créer à la mano.

```yaml
  - platform: template
    sensors:
      test_gitlab_projet_x_build_branch: #nom que l’on donne à notre sensor custom
        value_template: "{{ state_attr('sensor.test_gitlab_projet_x', 'build branch') }}" # On recupere et affiche l’attribute ‘build branche’
        friendly_name: "Branch"
    entity_id: test_gitlab_projet_x #Le sensor va écouter cet entity pour changer ses valeurs 
  - platform: template
    sensors:
      test_gitlab_projet_x_commit_date:
        value_template: "{{ state_attr('sensor.test_gitlab_projet_x', 'commit date') }}"
        friendly_name: "Date"
    entity_id: test_gitlab_projet_x

```

Ici `state_attr` nous permet de récupérer l’attribut 'commit date' du sensor gitlab.

Enfin, il nous reste plus qu’à configurer notre template pour afficher notre card

```yaml
#ui-lovelace.yaml

title: Dashboard
views:
    # View tab title.
  - title: Gitlab
    icon: mdi:gitlab
    id: gitlab
    cards:
      - type: entities
        title: Gitlab X Project
        entities:
          - entity: sensor.test_gitlab_projet_x
          - entity: sensor.test_gitlab_projet_x_build_branch
          - entity: sensor.test_gitlab_projet_x_commit_date
            icon: mdi:calendar-range
  - title: Other
    id: other

```

Tadaaa : 



Voilà pour la partie design, il y à des dizaines d’amélioration à apporter que nous ne verrons pas dans cet article.

Par exemple : 
- Afficher une card par branche (master, preprod, prod et dev)
- Changer la couleur de la card en fonction du test
- Raccourci vers le résultat de la ci
- Stats sur les ci
- …


## Interactions IRL


