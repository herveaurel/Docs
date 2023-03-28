# Home Assistant - Notification actionnable

✨Notifications avec choix cliquables ✨

## 1. Documentation 
1ere chose, la doc !
https://companion.home-assistant.io/.../actionable...
Sous ios, jusqu'a 10 choix dans une notif !
Plein d'autres options, voir la doc.

## 1. Mise en place
Le principe (sous ios) consiste a faire 1 automatisation pour definir l'action.

Exemple simple : Je veux switcher la lumiere de mon bureau via une notif. Auto necessaire pour connaitre cette action :

```
alias: test notif action allume bureau
description: ""
trigger:
- platform: event
event_type: ios.notification_action_fired
event_data:
actionName: ALLUME_BUREAU
condition: []
action:
- service: light.toggle
data: {}
target:
entity_id: light.lampe_de_bureau
mode: single
```

Une fois que ca c'est fait, on peut appeler cette action dans n'importe quelle automatisation, il suffit d'appeler le service notify.xxx et le remplir comme ceci (selon les options souhaitées) :

```
service: notify.mobile_app_iphone_aurel
data:
message: Allume bureau ?
title:  💡 Bureau
data:
actions:
- action: ALLUME_BUREAU
title: 🔦 On le switche ?
destructive: true
- action: RIEN
title: 🚫 Non
Vous souhaitez ajouter l'option 'critical" qui fera sonner votre tel ou votre montre malgré le mode silence et previendra d'une notif urgente ?
Ajouter ces lignes :
service: notify.mobile_app_iphone_aurel
data:
message: Allume bureau ?
title:  💡 Bureau
data:
push:
sound:
name: default
critical: 1
volume: 0.2
actions:
- action: ALLUME_BUREAU
title: 🔦 On le switche ?
destructive: true
- action: RIEN
title: 🚫 Non
```

![alt text](https://github.com/herveaurel/Docs/blob/main/Notification actionnable iOS/Captures/notif.jpg)

---------------------

## ⭐️ VIP 

Si vous aimez , likez 🌟 mon repo !

Si vous souhaitez m'offrir une petite bière 🍺 ou un café ☕️, dire merci 🙏, me soutenir ❤️‍🩹, ou m'encourager 💪🏼, et devenir VIP ⭐️ pour une aide personnalisée par message privé : https://www.paypal.com/paypalme/aaherve
Merci ! 
