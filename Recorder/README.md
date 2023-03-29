# Tutoriel utilisation du recorder de home assistant
Yann RITTER / Aurel RV Tuto Home Assistant 

Dans ce tutoriel tu trouveras des conseils d'optimisations de la base de données home assistant.

L'objectif est multiple : 
- Limitation des écritures en base de données donc préservation de ta carte SD
- Chargement plus rapide des graphiques 
- Reboot de home assistant plus rapide
- Plus de problèmes de backup trop lourd en échec


## Compatibilité
- Toutes versions de home assistant après la 0.7

## Source 
https://www.home-assistant.io/integrations/recorder/



## Avertissement 

Si, tu lis ça ! Non mais, Y’a pas que le tuto qui compte ! On a deux trois bricoles importantes à te dire avant.
Maintenant tu es adulte, t’as bien grandi, alors tu comprends que toutes les manipulations que tu vas faire sont sous ta responsabilité !
Si ton Home Assistant plante, ou que ton Raspberry explose… On n’y est pour rien !
Tout ce qui suit a été d’abord installé, testé par nos soins. Ça fonctionne. Et on l’utilise au quotidien !
Donc tu prends ton temps, tu lis bien, tu ne fais pas l’enfant à vouloir aller trop vite et sauter des étapes en mode « Ouais c’est bon je sais ! »
Ce n’est pas un tuto sur les bases, alors certaines étapes sont pour nous acquises et ne seront pas détaillées ici. Y’a toujours le groupe si vraiment tu as besoin d’aide…
On y va ?


## Prérequis

Avoir accès au fichier de configuration.yaml en utilisant l'un des deux outils ci-dessous
- SambaShare
- FileEditor

## Configuration du recorder
Tu vas devoir utiliser FileEditor ou sambaShare pour modifier ton fichier configuration.yaml

![alt text](https://github.com/ryann72/Home-assistant-tutoriel/blob/main/Recorder/Images/fichier.JPG)


Voici un exemple de notre configuration. Mais elle est à adapter selon tes besoins. 
Deux cas de figure :

-  Soit tu as utilisé notre précédent tutoriel pour l'aspirateur
https://github.com/ryann72/Home-assistant-tutoriel/blob/main/Aspirateur/tutoriel%20cartographie%20aspirateur.md 

Et on te conseille de supprimer les données de la carte de ta base de données en utilisant l'exemple ci-dessous :

````yaml recorder:
  purge_keep_days: 7 # durée de conservation des données
  exclude: # permet d'exclure tous les éléments listé ci-dessous de votre base de données
    domains: # filtrage par domaine
      - camera #supprime tous les entités caméra 
    entities: # filtrage par entités spécifiques
      - camera.aspirateur #supprime uniquement l'entité caméra de l'aspirateur
    event_types: # filtrage par evenements
      - call_service 
````
- Soit tu n'as pas de cartographie de l'aspirateur mais tu souhaites réduire ta base de données, voici un autre exemple :

Ajoute la configuration ci-dessous :
> Si tu as déjà « recorder : » tu ne le remets pas hein ! Tu colles en dessous…

````yaml recorder:
  purge_keep_days: 7 # durée de conservation des données
  exclude: # permet d'exclure tous les éléments listé ci-dessous de votre base de données
    domains: # filtrage par domaine
      - automation
      - updater
      - group
      - media_player
      - scene
      - script
      - sun
      - weather
    entities: # filtrage par entités spécifiques
      - sensor.last_boot 
      - sensor.date
      - camera.aspirateur #utile afin de ne pas surcharger votre base de donnée avec les coordonnées et carte de votre aspirateur chargé toutes les 5 secondes
    event_types: # filtrage par evenements
      - call_service 
````
Résultat : 

![alt text](https://github.com/ryann72/Home-assistant-tutoriel/blob/main/Recorder/Images/config1.JPG)

Une fois la configuration réalisée, il faut redémarrer home assistant.
A chaque modification apportée ici, un restart de Home Assistant est nécessaire.

Si tu veux encore réduire la taille, il est possible de récupérer les domaines que tu utilises

Dans les outils de développement colle le bloc suivant :
````
{%- for d in states | groupby('domain') %}
  {% if loop.first %}{{loop.length}} Domains:
  {% endif %}- {{ d[0] }}: {{d[0]|count}}
{%- endfor %}
````

Voici le resultat : 

![alt text](https://github.com/ryann72/Home-assistant-tutoriel/blob/main/Recorder/Images/domain2.jpg)

La liste des domaines affichés est utilisable afin de compléter la liste d'exclude.

## Retours persos
Depuis que nous utilisons cette méthode, nous avons énormément réduit la taille de la base de données
- Yann  de 1.2Go sur 1 journée à 80mo
- Aurel de 8Go sur 1 journée  à 50mo

Le résultat étant bien évidemment variable en fonction de ton utilisation, du nombre de sensors et du nombre d'éléments filtrés par le recorder.

Il y a également moins de lenteurs au démarrage / chargement des graphiques.

## Le mot de la fin
Une petite doc, un tuto, qui on espère t’a aidé le mieux possible.
Nous l’avons réalisé avec plaisir dans un esprit d’entre-aide, bénévolement, en parallèle de nos métiers et nos vies (si on en a !).
Nous n’assurons pas le support de tous les problèmes mais nous seront là si besoin sur le groupe, comme d’habitude.
Si tout fonctionne, remercie-moi.
Mais sinon, c’est forcément de sa faute.
Qui écrit ? Lui ou moi.
En ce moment ? Forcément moi, parfois lui.
Deux potes passionnés, Yann Ritter et Aurel RV.

-----

## ⭐️ Merci 

Si vous aimez , likez 🌟 mon repo !

Si vous souhaitez m'offrir une petite bière ou un café : https://www.paypal.com/paypalme/aaherve
Merci ! 