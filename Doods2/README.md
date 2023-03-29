# Home Assistant - Doods2 (par Aurel RV)


Je pars du principe que vous avez déjà au minimum une caméra fonctionnelle dans Home Assistant.
Sinon, la suite va être compliquée…


## 1. Création de caméras basées sur des photos : 

Tout d’abord, pour analyser des photos, il faut transformer ces photos en entités caméras, car doods demande des entités cameras. 
Donc je capture une image avec ma caméra de l’entrée et je l’enregistre avec le chemin : /media/detections/captures/entree_detection.jpg

C’est simple à faire avec platform: local_file, exemple : 

camera:
  - platform: local_file
    file_path: /media/detections/captures/entree_detection.jpg 
    name: entree_detection

Maintenant ma photo stockée dans Média est aussi une entité caméra nommée « camera.entree_detection »

-----

## 2. Installation de Doods2:

Doods peut détecter via des photos, via le flux live (personnellement non recommandé), et dispose de plusieurs labels de détections comme persons, car, cat, dog et beaucoup d’autres. 

La doc : https://www.home-assistant.io/integrations/doods/

![alt text](https://github.com/herveaurel/Docs/blob/main/Doods2/Captures/icon.png)

2 choses à faire pour installer doods2 : (voir liens dans la doc selon si Home Assistant add-on ou Docker container)

- Installer le module complémentaire (addon) avec ce dépôt : https://github.com/snowzach/hassio-addons , l’installation est longue. 

- Ces paramètres sont pour de l’analyse de photos, pour de la détection sur du flux il faudrait changer des paramètres. Ajouter ceci dans configuration.yaml : 

```
image_processing:
  - platform: doods
    url: "http://0.0.0.0:8080"
    detector: tensorflow #default
    scan_interval: 10000
    source:
      - entity_id: camera.entree_detection
      - entity_id: camera.entreealarme_detection
    file_out:
      - "/media/detections/verifications/{{ camera_entity.split('_')[0] }}/verifiee.jpg"
      - "/media/detections/verifications/{{ camera_entity.split('_')[0] }}/historique/{{ now().strftime('%Y%m%d') }}/{{ now().strftime('%H%M%S') }}.jpg"
    labels:
      - name: person
        confidence: 85
```

Dans cette configuration les entités des caméras sont les entités créés à base des photos comme vu juste avant. Pas des caméras réellement. 
Sous file_out, ce sont les deux chemins où s’enregistreront les détections réussies. 
- Le 1er pour le fichier écrasé à chaque fois
- Le 2e pour l’historique complet

Confidence : correspond au seuil en % à partir de combien la détection doit être validée. A changer selon. Vos tests. J’ai commencé à 40, je vais sûrement passer à 65-70. 

Après un reboot de HA, il y aura des entités « image_processing.doods_xxx » correspondants à chaque caméra listée dans la configuration ci-dessus, que nous utiliserons dans l’automatisation. 

## 3. Organisation des fichiers : 

Voici comment j’ai organisé ma bibliothèque Media. 
Tous les dossiers sont générés automatiquement, inutile d’aller les créer. 

Dans Média, j’ai un dossier « detections » 

Dedans il y a 2 sous dossiers : 
- captures : faites par les caméras 
- verifications : analyses par doods

Le dossier « verifications », comporte des sous dossiers générés automatiquement aussi, se nommant comme la camera utilisée. 
J’ai par ex un sous dossier « camera.entree » . 
Et ce dernier comporte un dossier « historique » avec toutes les détections réussies, et le fichier « verifiee.jpg » qui sert a être affiché sur le dashboard et être envoyé en notif, il est écrasé par chaque nouvelle détection réussie.

Le nettoyage de Media peut se faire manuellement (bon courage !) ou par shell_command, tous les 10 jours (au choix dans la commande), dans une automatisation. 

Dans le fichier configuration.yaml (ou shell_command.yaml si vous scindez comme moi) : 

shell_command:
    nettoyage_historiques_media: 'find /media/detections/verifications/camera.*/historique -depth -mindepth 1 -mtime +10 -delete'

Reboot de HA, puis créez une automatisation, comme par exemple : 

```
alias: Nettoyage historiques Media
description: ""
trigger:
  - platform: homeassistant
    event: start
  - platform: template
    value_template: "{{ is_state('sensor.date_jour', 'Lundi') }}"
condition: []
action:
  - service: shell_command.nettoyage_historiques_media
    data: {}
mode: single
```

-----

## 4. Automatisation: 

La logique va être de faire une capture, dans mon cas déclenchée par une porte, puis lancer le service d’analyse, et si le résultat est 0 c’est qu’il n’y a pas de détection humaine, sinon il y en a une. 
Doods enregistrera dans le dossier « historique » et créera le fichier vérification.jpg uniquement si il y a une détection positive. 

Pour info, le service qui va analyser juste après une capture photo, est : 
service: image_processing.scan
data: {}
target:
  entity_id: image_processing.doods_entree_detection


Une automatisation de base donnerai donc ceci : 


```
alias: Camera entrée snapshot détection sans alarme
description: ""
trigger:
  - platform: state
    entity_id:
      - binary_sensor.porte_entree
    to: "on"
    for:
      hours: 0
      minutes: 0
      seconds: 0
  - platform: state
    entity_id:
      - binary_sensor.porte_entree
    to: "off"
    for:
      hours: 0
      minutes: 0
      seconds: 0
condition:
  - condition: state
    entity_id: alarm_control_panel.alarme
    state: disarmed
action:
  - service: camera.snapshot
    target:
      entity_id: camera.camera_entree
    data:
      filename: /media/detections/captures/entree_detection.jpg
  - delay:
      hours: 0
      minutes: 0
      seconds: 0
      milliseconds: 500
  - service: image_processing.scan
    data: {}
    target:
      entity_id: image_processing.doods_entree_detection
mode: queued
max: 10
```


L’idéal est d’ajouter une option à la fin, et de recommencer l’opération 1, 2, 5, 10 fois ! avec un délai de 2 secondes. Afin que si aucune détection trouvée à la première photo, 2 sec de délai puis une photo est reprise et analysée, et sur cette dernière action, on recommence encore une fois ! 

Pour chaque caméras, comme celle de l’entrée pour continuer avec cet exemple, j’ai créé 2 caméras basées sur photos, pour quand l’alarme est activée, ou pas, afin de différenciés les images que j’affiche sur mon dashboard. A voir si cela vous est utile. 
J’ai donc 2 automatisations (je n’utilise pas les mêmes triggers), basés sur l’état de l’alarme, et donc chacune ne travaille pas avec les mêmes entités. 

Exemple d’automatisation complète si l’alarme est activée comportant 
- les reports si aucune détection, 
- un horodatage pour l’affichage sur le dashboard (géré avec un input text), 
- une notification push prioritaire et actionnable, 
- et l’envoi d’un enregistrement vidéo avec 30 secondes avant le déclencheur, en notification aussi 
(Tout ceci étant sauvegardé dans Média)


```
alias: Camera entrée snapshot détection avec alarme
description: ""
trigger:
  - platform: state
    entity_id:
      - binary_sensor.porte_cagibi
      - binary_sensor.porte_entree
    to: "on"
    for:
      hours: 0
      minutes: 0
      seconds: 0
  - platform: state
    entity_id:
      - binary_sensor.porte_entree
      - binary_sensor.porte_cagibi
    to: "off"
    for:
      hours: 0
      minutes: 0
      seconds: 0
condition:
  - condition: not
    conditions:
      - condition: state
        entity_id: alarm_control_panel.alarme
        state: disarmed
      - condition: state
        entity_id: alarm_control_panel.alarme
        state: arming
action:
  - service: camera.snapshot
    target:
      entity_id: camera.camera_entree
    data:
      filename: /media/detections/captures/entree_alarme_detection.jpg
  - delay:
      hours: 0
      minutes: 0
      seconds: 0
      milliseconds: 500
  - service: image_processing.scan
    data: {}
    target:
      entity_id: image_processing.doods_entreealarme_detection
  - repeat:
      count: "10"
      sequence:
        - if:
            - condition: numeric_state
              entity_id: image_processing.doods_entreealarme_detection
              below: 1
          then:
            - delay:
                hours: 0
                minutes: 0
                seconds: 2
                milliseconds: 0
            - service: camera.snapshot
              target:
                entity_id: camera.camera_entree
              data:
                filename: /media/detections/captures/entree_alarme_detection.jpg
            - delay:
                hours: 0
                minutes: 0
                seconds: 0
                milliseconds: 500
            - service: image_processing.scan
              data: {}
              target:
                entity_id: image_processing.doods_entreealarme_detection
          else:
            - service: input_text.set_value
              data:
                value: "{{ now().strftime(\"%d/%m - %H:%M\") }}"
              target:
                entity_id: input_text.horodatage_cam_entree_alarme
            - service: notify.mobile_app_iphone_aurel
              data:
                message: "Personne détectée ! "
                title: Entrée 📸
                data:
                  image: /media/local/detections/captures/entree_alarme_detection.jpg
                  push:
                    sound:
                      name: default
                      critical: 1
                      volume: 0
                  actions:
                    - action: DESACTIVER_ALARME
                      title: 🏠 Désactiver l'alarme
                      destructive: true
                    - action: ACTIVER_ALARME
                      title: 🚨 Activer l'alarme
                      destructive: false
                    - action: RIEN
                      title: Annuler
            - service: camera.record
              data:
                lookback: 30
                filename: /media/detections/captures/detection.mp4
                duration: 20
              target:
                entity_id: camera.camera_entree
            - delay:
                hours: 0
                minutes: 0
                seconds: 25
                milliseconds: 0
            - service: notify.mobile_app_iphone_aurel
              data:
                message: "Vidéo ! "
                title: Entrée 🎬
                data:
                  video: /media/local/detections/captures/detection.mp4
                  push:
                    sound:
                      name: default
                      critical: 1
                      volume: 0
                  actions:
                    - action: DESACTIVER_ALARME
                      title: 🏠 Désactiver l'alarme
                      destructive: true
                    - action: ACTIVER_ALARME
                      title: 🚨 Activer l'alarme
                      destructive: false
                    - action: RIEN
                      title: Annuler
            - service: shell_command.copy_entree_video
              data: {}
            - stop: Détection réussie
mode: single
```

Autre exemple d’automatisation complète si l’alarme est désactivée comportant 
- les reports si aucune détection, 
- un horodatage pour l’affichage sur le dashboard, 
- une notification push prioritaire et actionnable uniquement si je ne suis pas à mon domicile
(Tout ceci étant sauvegardé dans Média)

```
alias: Camera entrée snapshot détection sans alarme
description: ""
trigger:
  - platform: state
    entity_id:
      - binary_sensor.porte_entree
    to: "on"
    for:
      hours: 0
      minutes: 0
      seconds: 0
  - platform: state
    entity_id:
      - binary_sensor.porte_entree
    to: "off"
    for:
      hours: 0
      minutes: 0
      seconds: 0
condition:
  - condition: or
    conditions:
      - condition: state
        entity_id: alarm_control_panel.alarme
        state: arming
      - condition: state
        entity_id: alarm_control_panel.alarme
        state: disarmed
action:
  - service: camera.snapshot
    target:
      entity_id: camera.camera_entree
    data:
      filename: /media/detections/captures/entree_detection.jpg
    enabled: true
  - delay:
      hours: 0
      minutes: 0
      seconds: 0
      milliseconds: 500
    enabled: true
  - service: image_processing.scan
    data: {}
    target:
      entity_id: image_processing.doods_entree_detection
    enabled: true
  - repeat:
      count: "10"
      sequence:
        - if:
            - condition: numeric_state
              entity_id: image_processing.doods_entree_detection
              below: 1
          then:
            - delay:
                hours: 0
                minutes: 0
                seconds: 2
                milliseconds: 0
            - service: camera.snapshot
              target:
                entity_id: camera.camera_entree
              data:
                filename: /media/detections/captures/entree_detection.jpg
            - delay:
                hours: 0
                minutes: 0
                seconds: 0
                milliseconds: 500
            - service: image_processing.scan
              data: {}
              target:
                entity_id: image_processing.doods_entree_detection
          else:
            - service: input_text.set_value
              data:
                value: "{{ now().strftime(\"%d/%m - %H:%M\") }}"
              target:
                entity_id: input_text.horodatage_cam_entree
            - if:
                - condition: not
                  conditions:
                    - condition: state
                      entity_id: person.herve
                      state: home
              then:
                - service: notify.mobile_app_iphone_aurel
                  data:
                    message: "Personne détectée ! "
                    title: Entrée 📸
                    data:
                      image: /media/local/detections/captures/entree_detection.jpg
                      push:
                        sound:
                          name: default
                          critical: 0
                          volume: 0
                      actions:
                        - action: ACTIVER_ALARME
                          title: 🚨 Activer l'alarme
                          destructive: true
                        - action: DESACTIVER_ALARME
                          title: 🏠 Désactiver l'alarme
                          destructive: false
                        - action: RIEN
                          title: Annuler
                - stop: Détection réussie
              else:
                - stop: Détection réussie
mode: single
```

Libre à vous de modifier vos déclencheurs, vos actions, mais après de nombreux tests et plusieurs jours, je ne peux que vous conseiller ce principe de détection et de report.

---------------------

## 5. Infos: 

Les détections sont dans un encadré, et en zoomant, au dessus en petit, on peut voir le niveau de détection (confidence) en %. 

![alt text](https://github.com/herveaurel/Docs/blob/main/Doods2/Captures/confidence.png)


Pour ce type de carte vous pouvez vous référer directement à mon GitHub : https://github.com/herveaurel/HomeAssistant 

![alt text](https://github.com/herveaurel/Docs/blob/main/Doods2/Captures/carte.jpg)

---------------------

## ⭐️ VIP 

Si vous aimez , likez 🌟 mon repo !

Si vous souhaitez m'offrir une petite bière ou un café : https://www.paypal.com/paypalme/aaherve
Merci ! 
