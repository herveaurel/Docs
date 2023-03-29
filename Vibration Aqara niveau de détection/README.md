# Home Assistant - Capteur de vibration Aqara : modifier le niveau de détection

## ZHA - Clusters
Savez-vous qu’il est possible grâce à la puissance de ZHA et le menu des clusters de modifier plein de choses ? 

## Tutoriel
Par exemple un capteur vibrations Aqara. 
Plusieurs sensibilités de vibrations sont possibles quand on possède le hub Aqara. 

Et bien avec ZHA aussi ! Et voici comment : 

- Sous configuration -> ZHA
- Sélectionne ton device, sûrement LUMI lumi.vibration.aq1 si tu ne l’as pas renommé 
- ouvre le menu des clusters
- Choisis VibrationBasicCluster (in)
- Sélectionnez ensuite l'attribut de sensibilité
- Mets 1 pour la valeur et 4447 pour le code fabriquant 
- Ensuite, valide : définir l'attribut ZIgbee

```
1 : très sensible 
11 : moyen 
21 : peu sensible 
```

Et roule ma poule ! 



---------------------

## ⭐️ Merci 

Si vous aimez , likez 🌟 mon repo !

Si vous souhaitez m'offrir une petite bière ou un café : https://www.paypal.com/paypalme/aaherve
Merci ! 