**Pour chaque formulaire créer un objet (non relié à active record) specifique au
formulaire ex: AddressForm**


### Page d'accueil
   **url**:  /<br>
   La page doit etre transformée afin d'être générique<br>
   Il faut donc adapter les textes "Nos depanneurs en 3 clics..."<br>
   Il faut ajouter **3 Boutons avec image** : Serrurerie / Plomberie / Vitrerie
   les boutons menent directement sur la **page profession**


### Page profession
 **url**: /plomberie /serrurie /vitrerie <br>
 Affche les boutons avec image des interventions: Porte claquée / Vitre cassée
 Les boutons menent directement sur la **page catégorie**


### Page catégorie
 **url**: /plomberie/fuite, /serrurie/porte-claquee<br>
  Affiche un formulaire pour rentrer l'addresse<br>
  **Si formulaire OK** On crée une adresse et on stocke l'id de l'adresse en session<br>
  **Contrainte**:
  - Si on retourne sur cette page et qu'on a une adresse en session : Le formulaire est prerempli & la validation met à jour l'adresse courante<br>
  - Si on retourne sur cette page et qu'on a un customer en session & une intervention en cours on previent qu'il est parti pour recommander on prerempli le form avec **l'adresse du customer**
  - Si le code postal n'est pas supporté retourner sur le formulaire
  **La validation de la page renvoie sur la page devis**


### Page devis
**url**: /devis/:id (avec id celui de l'adresse)<br>
Affiche Bloc récapitulatif du devis:
- On affiche le nombre de depanneur disponible
- Affiche le prix
- Affiche le temps de route (on met toujours 30min)
- Affiche formulaire (nom, prénom, email)

**Si Form OK**
 - Met à jour l'addresse
 - On crée le customer si non présent en session
 - On associe un duplicata de l'adresse au customer (ids différents pour customer & intervention)
 - On crée une intervention du bon type (pending_pro_validation)
 - On associe le customer à l'intervention
 - On associe l'adresse à l'intervention
 - On supprime l'adresse de la session
 - On met en session l'id du customer (si il n'est pas déjà présent)
 - On envoie un mail client (intervention) + un sms client (intervention)
 - On envoie un mail client (création de compte) si le client vient d'être crée
 - On envoie un mail pro  (demande d'intervention )+ un sms pro (demande d'intervention)
 -
 **Contrainte**:
- Si pas d'addresse en session on redirige sur la page d'accueil

**La validation envoie sur la page de recap de commande**

### Page recap commande
**url**: /commande/id<br>
 - Texte en fonction de l'état de l'intervention
 - Bloc recapitualitif de la commande avec prix, adresse & état du depanneur<br>
 - Ajouter un bouton qui mene vers les interventions (/interventions)
**Contrainte**:
 - Verifier que l'intervention appartient bien à la personne (customer en session, id commande dans l'url)

### Page interventions
**url**: /interventions
- Liste l'ensemble des interventions avec l'état (comme actuellement)
- Ajouter le prix<br>
**Contrainte**:
- Si il n'y a pas de customer en session alors on redirige sur la page d'accueil
