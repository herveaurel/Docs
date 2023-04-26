# Home Assistant - Switchbot Lock (par Aurel RV)

SwitchBot m'a proposé de tester la SwitchBot Lock, avec le Keypad Touch (code, emprunte digitale et NFC), et le hub mini. 

Avoir les serrures connectées et automatisées s'est révélé être un avantage précieux et il serait difficile de revenir en arrière maintenant ! 


## 1. Présentation : 

![alt text](https://github.com/herveaurel/Docs/blob/main/SwitchBot%20Lock/Captures/presentation.jpg)


Je ne vais pas vous présenter le produit dans sa globalité, il y a suffisamment d'articles et de vidéos très complètes sur internet. 

Il me semble important tout de même de vous préciser que :

- déclinée en 2 coloris, noir et gris 
- ce produit fonctionne en Bluetooth
- votre clé d'origine reste dans le barillet, la serrure fixée par un autocollant, englobe la clé
- le Keypad se fixe à l'extérieur et le modèle que j'ai reçu dispose des options par code, emprunte digitale et NFC : chaque utilisateur peut avoir son propre code et plusieurs empruntes. 
- le rôle du hub mini est d'avoir le contrôle à distance et les notifications via l'application SwitchBot mais Home Assistant permet de s'en passer 

Prenez le temps de lire les documentations avant d'investir, car il peu y avoir des recommandations importantes afin de vérifier la compatibilité de votre porte. 


![alt text](https://github.com/herveaurel/Docs/blob/main/SwitchBot%20Lock/Captures/gris.JPG)

![alt text](https://github.com/herveaurel/Docs/blob/main/SwitchBot%20Lock/Captures/noir.JPG)

![alt text](https://github.com/herveaurel/Docs/blob/main/SwitchBot%20Lock/Captures/keypad.JPG)

-----

## 2. Home Assistant :

⚠️ L'intégration SwitchBot ne prend pas en charge les comptes SSO (Connexion avec Google, etc.), mais uniquement les comptes de nom d'utilisateur et de mot de passe.

L'intégration SwitchBot découvrira automatiquement les appareils une fois que l'intégration Bluetooth sera activée et fonctionnelle.

➡️ Documentation officielle : https://www.home-assistant.io/integrations/switchbot/


En effet, l'intégration m'a proposé des découvertes rapidement, mais certaines complètement inutiles. 

Les serrures remontent parfaitement, quant aux Keypad et le Hub Mini, les remontées sont inutiles, et inutilisables. 

Exemple des remontées pour un Keypad : 

![alt text](https://github.com/herveaurel/Docs/blob/main/SwitchBot%20Lock/Captures/remontee_keypad.jpg)

Ou pour le hub mini : 

![alt text](https://github.com/herveaurel/Docs/blob/main/SwitchBot%20Lock/Captures/remontee_hub.jpg)

Ce qui nous intéresse, les serrures : 

![alt text](https://github.com/herveaurel/Docs/blob/main/SwitchBot%20Lock/Captures/remontee_lock.jpg)

Là, c'est simple, accès aux commandes verrouiller / déverrouiller, et si vous avez fixé le capteur de porte fourni, il remonte bien également. 

La remontée de la pile est également de la partie. 

## 3. Automatisations : 

Certainement la partie la plus importante afin de rendre ce produit et notre logement encore plus intelligent. 

Au fil des semaines j'ai construit plusieurs programmes afin de répondre a divers besoin. 

Voici des exemples de programmations : 



A.  Script de verrouillage (un autre identique pour le déverrouillage) et de debug si la serrure ne répond pas  : 

```
alias: Ferme l'entrée
mode: restart
sequence:
  - if:
      - condition: state
        entity_id: binary_sensor.porte_entree
        state: "off"
      - condition: state
        entity_id: lock.serrure_entree
        state: unlocked
    then:
      - service: lock.lock
        data: {}
        target:
          entity_id: lock.serrure_entree
      - wait_for_trigger:
          - platform: state
            entity_id:
              - lock.serrure_entree
            to: locked
        continue_on_timeout: true
        timeout:
          hours: 0
          minutes: 0
          seconds: 10
          milliseconds: 0
      - condition: not
        conditions:
          - condition: state
            entity_id: lock.serrure_entree
            state: locked
      - service: homeassistant.reload_config_entry
        data: {}
        target:
          entity_id: lock.serrure_entree
      - repeat:
          until:
            - condition: state
              entity_id: lock.serrure_entree
              state: locked
          sequence:
            - service: lock.lock
              data: {}
              target:
                entity_id: lock.serrure_entree
            - delay:
                hours: 0
                minutes: 0
                seconds: 30
                milliseconds: 0
    else:
      - if:
          - condition: state
            entity_id: lock.serrure_entree
            state: locked
        then:
          - stop: Serrure déjà fermée
        else:
          - if:
              - condition: state
                entity_id: binary_sensor.porte_entree
                state: "on"
            then:
              - service: notify.notify
                data:
                  title: ⚠️ Serrure entrée
                  message: "Verrouillage de l'entrée impossible, la porte est ouverte. "
                  data:
                    actions:
                      - action: FERMER_ENTREE
                        title: Essayer de vérrouiller l'entrée à nouveau
                        destructive: true
                      - action: rien
                        title: Non
            else:
              - service: notify.notify
                data:
                  title: ⚠️ Serrure entrée
                  message: >-
                    Verrouillage de l'entrée impossible, la serrure est
                    coincée. 
                  data:
                    actions:
                      - action: FERMER_ENTREE
                        title: Essayer de vérrouiller l'entrée à nouveau
                        destructive: true
                      - action: OUVRIR_ENTREE
                        title: Essayer de dévérrouiller l'entrée à nouveau
                        destructive: true
                      - action: rien
                        title: Non
icon: mdi:lock

```


B.  Script d'auto switch verrouiller / déverrouiller  : 

```
alias: "entrée serrure toggle "
sequence:
  - if:
      - condition: state
        entity_id: lock.serrure_entree
        state: locked
    then:
      - service: script.ouvre_lentree
        data: {}
    else:
      - service: script.ferme_lentree
        data: {}
mode: parallel
icon: mdi:lock
max: 10 

```



C. L'alarme est désarmée, je recois une notification avec choix actionnable à l'approche de mon domicile : 

```
alias: Serrure notif si aurel arrive
description: ""
trigger:
  - entity_id: person.herve
    event: enter
    platform: zone
    zone: zone.home
condition:
  - condition: state
    entity_id: alarm_control_panel.alarme
    state: disarmed
  - condition: state
    entity_id: group.serrures
    state: locked
action:
  - service: notify.mobile_app_iphone_aurel
    data:
      message: Quelle serrure ?
      title: Déverrouiller la maison ?
      data:
        actions:
          - action: OUVRIR_ENTREE
            title: Ouvrir l'entrée
            destructive: true
          - action: OUVRIR_GARAGE
            title: Ouvrir le garage
            destructive: true
          - action: rien
            title: Aucune
        group: serrure
        tag: ouvrir_serrure
  - wait_for_trigger:
      - platform: state
        entity_id:
          - lock.serrure_entree
          - lock.garage
        to: unlocked
      - platform: state
        entity_id:
          - binary_sensor.porte_entree
          - binary_sensor.porte_garage
        to: "on"
      - platform: template
        value_template: "{{not is_state('person.herve', 'home')}}"
    continue_on_timeout: false
  - service: notify.mobile_app_iphone_aurel
    data:
      message: clear_notification
      data:
        tag: ouvrir_serrure
mode: single
```


D.  Verrouillage automatique si je suis présent : 

```
alias: Serrure entrée verrouillage auto si présent
description: ""
trigger:
  - platform: state
    entity_id:
      - binary_sensor.porte_entree
    to: "off"
    for:
      hours: 0
      minutes: 15
      seconds: 0
  - platform: state
    entity_id:
      - lock.serrure_entree
    to: unlocked
    for:
      hours: 0
      minutes: 15
      seconds: 0
  - platform: time_pattern
    minutes: /15
condition:
  - condition: state
    entity_id: input_boolean.serrure_entree_verrouillage_automatique
    state: "on"
  - condition: state
    entity_id: lock.serrure_entree
    state: unlocked
  - condition: state
    entity_id: binary_sensor.porte_entree
    state: "off"
  - condition: state
    entity_id: binary_sensor.serrure_entree_porte
    state: "off"
  - condition: template
    value_template: >-
      {{as_timestamp(now()) -
      as_timestamp(states.lock.serrure_entree.last_changed) > 870 and
      as_timestamp(now()) -
      as_timestamp(states.binary_sensor.porte_entree.last_changed) > 870 }}
  - condition: or
    conditions:
      - condition: state
        entity_id: person.alex
        state: home
      - condition: state
        entity_id: person.herve
        state: home
action:
  - service: script.ferme_lentree
    data: {}
mode: single

```


E.  Verrouillage automtique si je suis absent : 

```
alias: "Serrure entrée verrouillage auto si absent "
description: ""
trigger:
  - platform: state
    entity_id:
      - binary_sensor.porte_entree
    to: "off"
    for:
      hours: 0
      minutes: 2
      seconds: 0
  - platform: state
    entity_id:
      - lock.serrure_entree
    to: unlocked
    for:
      hours: 0
      minutes: 2
      seconds: 0
  - platform: state
    entity_id:
      - person.alex
    from: home
  - platform: state
    entity_id:
      - person.herve
    from: home
  - platform: time_pattern
    minutes: /5
condition:
  - condition: state
    entity_id: input_boolean.serrure_entree_verrouillage_automatique
    state: "on"
  - condition: state
    entity_id: lock.serrure_entree
    state: unlocked
  - condition: state
    entity_id: binary_sensor.porte_entree
    state: "off"
  - condition: state
    entity_id: binary_sensor.serrure_entree_porte
    state: "off"
  - condition: template
    value_template: >-
      {{as_timestamp(now()) -
      as_timestamp(states.lock.serrure_entree.last_changed) > 870 and
      as_timestamp(now()) -
      as_timestamp(states.binary_sensor.porte_entree.last_changed) > 870 }}
  - condition: not
    conditions:
      - condition: state
        entity_id: person.alex
        state: home
      - condition: state
        entity_id: person.herve
        state: home
action:
  - service: script.ferme_lentree
    data: {}
mode: single

```

F.  Si la serrure est coincée (et oui ca peut arriver ! ) : 

```

alias: Serrure entrée coincée
description: ""
trigger:
  - platform: state
    entity_id:
      - lock.serrure_entree
    to: jammed
  - platform: state
    entity_id:
      - lock.serrure_entree
    from: jammed
condition: []
action:
  - if:
      - condition: state
        entity_id: lock.serrure_entree
        state: jammed
    then:
      - service: notify.mobile_app_iphone_aurel
        data:
          title: ⚠️ Serrure entrée
          message: "La serrure est coincée, vérifiez la ! "
          data:
            actions:
              - action: FERMER_ENTREE
                title: Vérrouiller l'entrée
                destructive: true
              - action: rien
                title: Non
            group: serrure
            tag: serrure_entree_coincee
      - if:
          - condition: state
            entity_id: person.alex
            state: home
        then:
          - service: notify.mobile_app_iphone_alex
            data:
              title: ⚠️ Serrure entrée
              message: "La serrure est coincée, vérifiez la ! "
              data:
                actions:
                  - action: FERMER_ENTREE
                    title: Essayez de re-vérrouiller
                    destructive: true
                  - action: rien
                    title: Je vérifie moi-même
                group: serrure
                tag: serrure_entree_coincee
    else:
      - service: notify.notify
        data:
          message: clear_notification
          data:
            tag: serrure_entree_coincee
mode: single

```


Sans oublier d'inclure les commandes de verrouillage dans l'automatisation qui arme l'alarme par exemple. 


-----

## 4. Tableau de bord : 

Libre à chacun d'utiliser ou de créer ses propres cartes, je vous montre ma réalisation, que vous pouvez retrouver sur mon Github : https://github.com/herveaurel/HomeAssistant 

![alt text](https://github.com/herveaurel/Docs/blob/main/SwitchBot%20Lock/Captures/dashboard.jpg)


---------------------

## 5. Conclusion : 

Je me sens bien plus en sécurité ayant maintenant ces serrures sur les deux portes principales de mon logement. 

Que ce soit le verrouillage automatique qui permet d'avoir une maison toujours fermée à clé, ou le déverrouillage simple et rapide, à distance ou via le Keypad, adultes et enfants y trouvent leur compte et satisfaction. 

Via l'application SwitchBot, sur smartphone ou montre connectée, la réactivité de la serrure est quasi instantanée. 
Hélas ce n'est pas le cas via Home Assistant, totalement aléatoire, pouvant être instantanée comme prendre presque 1 minute. J'ai même parfois dépasser la minute chronomètre en main. C'est pourquoi j'ai créé le script de debug, qui réduit quand même pas mal cette latence. 

Au début j'utilisais les possibilités de l'application SwitchBot, mais, ayant peaufiné mes programmations Home Assistant, ces dernières suffisent. 

Dommage que les Keypad ne remontent pas, j'aurais adoré avoir des infos comme avec d'autres marques, à savoir " déverrouillé via le keypad", ou "qui a déverrouillé avec le Keypad" suivant le code tapé par exemple... 



---------------------

## ⭐️ Merci 

Si vous aimez , likez 🌟 mon repo !

Si vous souhaitez m'offrir une petite bière ou un café : https://www.paypal.com/paypalme/aaherve
Merci ! 
