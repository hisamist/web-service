# Études de cas — Groupe : ..............

## Cas 1
1. Où est le problème ? Décrivez-le en une phrase, comme à un collègue.
- Je trouve que les chemins a de problème comme nommage irregulier(camelcase,snakecase),certains routes inclutent les verbes,delete par POST, GET methode sans utiliser DELETE methode, et manque d'interface uniforme. 
2. Quel dégât concret : prix, sécurité ou scalabilité ? Donnez un exemple chiffré ou une situation.
- **Scalabilité et prix** : `GET /students` ou `/notes_list` sans pagination renvoie les 5 000 éléments à chaque appel. À environ 1 Ko par élément, cela fait 5 Mo par appel ; avec 1 000 appels par jour, on transfère 5 Go par jour pour rien. La facture de la base cloud (Supabase) augmente, les temps de réponse aussi, et la situation empire à chaque nouvel étudiant.
- **Sécurité** : une suppression déclenchée par un GET peut être lancée par accident (voir question 5).
- **Compatibilité** : si on corrige les noms des routes du jour au lendemain, toutes les applis qui utilisent déjà l'API cassent. Il faut garder les anciennes routes le temps de la migration, ou publier les nouvelles sous `/v2`.
3. Quel principe du cours s'applique ?
- 1 Des noms dans l'URL, l'action dans le verbe
- 2 Le code HTTP dit la vérité(Utilise GET/POST pour DELETE envoie 200 mais cela pas correspond pas la vérité?)
- 3 Jamais de liste sans limite 
- On ajoute, on ne casse pas( Pour mettre à jour de API)
5. La dernière ligne cache un danger particulier. Lequel ?
  - /getStudentNotes?student=4&delete=true expose ID de student et aussi expose params delete boolean et logique delete interieur.
**Notre correction :**
   GET  /students
   DELETE /students/4
   GET /notes
   POST /notes
   GET /students/4/notes

## Cas 2 La place F7
1.  Il envoie 200 avec un message d'erreur pour POST booking, mais il faut plutôt envoyer 409 Conflict, 
parce que le code HTTP dit la vérité en tant que convention et pour éviter les contradictions dans 
l'application, et toujours refléter les vrais états réels.
2.  L'utilisateur croit que sa réservation a réussi parce que l'app reçoit 200 et affiche « réussi »sans message d'erreur. Résultat : ce n'est pas le cas, il n'a pas pu réserver, ce qui coûte une baisse du chiffre d'affaires et une perte de clients au cinéma. Les spectateurs arrivent dans la salle et il y a une queue, parce qu'une personne ne peut pas trouver sa place à cause d'un double booking sur le site.
3. Le code HTTP dit la vérité
4. HTTP/1.1 409 Conflict
Content-Type: application/json
{ "movieId": 12, "seat": "F7" , "message": "Already reserved this seat."}
5. Un conflit de réplication (souvent associé à un code HTTP 409 ou une erreur applicative E42) se produit lorsque deux utilisateurs ou appareils modifient le même enregistrement hors ligne avant de se synchroniser.
C'est que le developpeur d'app qui doit savoir. 


## Cas 3 — La page d'accueil met 9 secondes

**1. Où est le problème ?**

Il envoie tous les 80 000 produits juste pour afficher 12 items sur le premier écran, et envoie aussi les données depuis la DB sans DTO, ce qui expose toutes les infos.

**2. Quel dégât concret ?**

- Envoyer 80 000 items dont 12 utilisés peut augmenter le prix de la DB cloud (Supabase), baisser la performance, et aussi bloquer la scalabilité pour gérer plus de données avec plus de clients.
- Envoyer toutes les données des produits directement depuis la DB sans DTO peut exposer trop d'infos, cela augmente les risques d'attaque contre la DB.

**3. Quel principe du cours s'applique ?**

- **Principe 3** : Jamais de liste sans limite
- **Principe 5** : N'envoyer que ce que l'écran affiche

**4. La requête du front et la réponse de l'API**

*Requête du front :*

```http
GET /v1/products?limit=12&offset=0
```

*Réponse de l'API :*

```json
{
  "data": [
    {
      "id": "101",
      "name": "Casque audio sans fil",
      "price": 129.99,
    } ...
    // 12 data au total
  ],
  "pagination": {
    "limit": 12,
    "offset": 0,
    "total": 80000
  }
}
```

*Backend :*

```js
function toProductDto(p) {
  return {
    id: p.id,
    name: p.name,
    price: p.price,
  };
}

app.get('/v1/products', async (req, res) => {
  const limit = Number(req.query.limit) || 12;

  const products = await db.query(
    'SELECT id, name, price FROM products LIMIT $1',
    [limit]
  );

  res.json(products.map(toProductDto));
});
```

**5. Le front demande 500 000 produits d'un coup. Que fait votre API ?**

Envoyer BAD Request 400, 422

## Cas 4 Trois dons de 20 €
1.L'endpoint POST /v1/donations n'est pas idempotent (pas de clé d'idempotence ni de vérification de doublon) et déclenche un débit bancaire à chaque tentative, ce qui fait que les réessais réseau de l'appli créent plusieurs dons et plusieurs débits de 20 €.
2.Les payments inattendu pour l'utilisateur, il faut remboursement, risque de perte de client, basse de qualité de service. 
3.Un compteur à l'entrée
4.Décrivez la solution étape par étape : que fait l'appli, que fait le serveur au premier envoi, que fait-il au deuxième ?
### 4. Solution étape par étape
* **Ce que fait l'appli :** elle crée une clé unique (clé d'idempotence) pour ce don et l'envoie avec la requête. En cas de coupure réseau et de réessai, elle **garde exactement la même clé**.
* **Ce que fait le serveur au 1er envoi :** il enregistre la clé comme « en cours », crée le don, fait le débit bancaire, sauvegarde le résultat, puis renvoie **`201`** avec le don.
* **Ce que fait le serveur au 2e envoi (avec la même clé) :**
  * Si c'est **déjà terminé**, il **ne redébite pas** et renvoie directement le résultat déjà enregistré (`201`).
  * Si c'est **encore en cours**, il **ne redébite pas** non plus et indique que le traitement est déjà en cours.
    ```
    app.post('/v1/donations', async (req, res) => {
    const key = req.header('Idempotency-Key');
    const userId = req.user.id;

    if (!key) {
        return res.status(400).json({ error: 'Idempotency-Key header is required' });
    }

    // 1. Vérifier si cette clé existe déjà pour cet utilisateur
    const existing = await db.idempotencyKeys.findOne({ key, userId });

    if (existing) {
        if (existing.status === 'COMPLETED') {
        // Déjà traité : renvoyer la même réponse sans redébiter
        return res.status(201).json(existing.response);
        }
        // Encore en cours de traitement
        return res.status(409).json({
        error: 'REQUEST_IN_PROGRESS',
        message: 'Traitement déjà en cours pour cette requête.',
        });
    }

    // 2. Marquer la clé comme "en cours" avant l'action sensible
    await db.idempotencyKeys.insert({
        key,
        userId,
        status: 'PROCESSING',
        response: null,
    });

    try {
        // 3. Créer le don et débiter la carte une seule fois
        const don = await db.donations.insert({
        cagnotteId: req.body.cagnotteId,
        montant: req.body.montant,
        userId,
        });

        await banque.debiter(req.user.carte, req.body.montant);

        // 4. Sauvegarder le résultat final et renvoyer 201
        await db.idempotencyKeys.update(
        { key, userId },
        { status: 'COMPLETED', response: don }
        );

        return res.status(201).json(don);
    } catch (err) {
        // En cas d'échec, libérer ou marquer la clé en erreur pour permettre un vrai réessai
        await db.idempotencyKeys.update(
        { key, userId },
        { status: 'FAILED' }
        );
        throw err;
    }
    });
    ```

5. Regardez l'ordre des deux opérations dans le code. Que se passe-t-il si le débit échoue ?
Avec le premier code, il ne fait rien. Le code proposé par 4, on va mettre à jour de l'état de 
clé d'idempotent. 

## Cas 5 La fiche du conducteur

**1. Où est le problème ? Décrivez-le en une phrase, comme à un collègue.**

Il envoie les informations privées importantes comme l'IBAN, le password.

**2. Quel dégât concret : prix, sécurité ou scalabilité ? Donnez un exemple chiffré ou une situation.**

Cela amène une grande fuite d'informations de clients et une mauvaise utilisation. Il faut compenser les victimes, et aussi baisse de crédibilité du service, et perte de clients.

**3. Quel principe du cours s'applique ?**

N'envoie pas ce que l'écran affiche

**4. Classez les 8 champs en trois colonnes : visible par tous, réservé au conducteur lui-même, ne sort jamais de l'API.**

| Visible par tous | Réservé au conducteur lui-même | Ne sort jamais de l'API |
|------------------|--------------------------------|-------------------------|
| `id`             | `phone`                        | `passwordHash`          |
| `firstName`      | `homeAddress`                  | `iban`                  |
| `rating`         | `licenseNumber`                |                         |

**5. « Ce n'est pas grave, le front ne les affiche pas. » Que répondez-vous ?**

Dès l'instant où une donnée est incluse dans la réponse JSON, elle est déjà partie sur le réseau : même si l'interface ne l'affiche pas, n'importe qui peut facilement la voir via les outils de développement du navigateur (onglet Réseau), curl ou Postman.

## Cas 6 La facture de mars

1. Où est le problème ? Décrivez-le en une phrase, comme à un collègue.
L'endpoint GET /v1/forecast est ouvert sans authentification ni limitation, n'importe quoi peut acceder et appelle un fournisseur externe payant à chaque requête sans mise en cache, ce qui provoque une explosion des appels quand il y a trop d'access. 
2. Quel dégât concret : prix, sécurité ou scalabilité ? Donnez un exemple chiffré ou une situation.
Catégorie principale : Prix (et scalabilité / dépendance externe).
Exemple chiffré : à 0,002 € par appel :
10 000 appels/jour = 20 €/jour (600 €/mois).
Multiplié par 40 (400 000 appels/jour) = 800 €/jour (24 000 €/mois) rien que pour un seul endpoint, sans compter le risque d'atteindre les quotas ou latences du fournisseur externe.
3. Quel principe du cours s'applique ?
**Un compteur à l'entrée**, parce que chaque client doit avoir une limite, et qu'au-delà l'API doit répondre `429` **avant** d'appeler la base ou le fournisseur payant.
**Que mettez-vous en place ? Précisez : qui est compté, quelle limite, et ce que l'API répond quand elle est dépassée.**
4. Que mettez-vous en place ? Précisez : qui est compté, quelle limite, et ce que l'API répond quand elle est dépassée.
- Qui est compté : l'adresse IP du client (ou une clé d'API si on identifie les partenaires).
- Quelle limite : par exemple 60 requêtes par minute par adresse IP (un seuil cohérent avec un usage normal, mais qui bloque le polling à 1 requête/seconde).

- Ce que l'API répond quand la limite est dépassée :
Le code HTTP 429 Too Many Requests.
L'en-tête Retry-After (ex. Retry-After: 60) indiquant dans combien de secondes réessayer.
Un message JSON clair expliquant que le quota est dépassé, sans appeler le fournisseur (donc 0 € facturé).
5. Vous limitez par adresse IP. Une école de 800 étudiants sort sur Internet avec une seule adresse. Que se passe-t-il ?
Ils sont consideré de meme adress et seront bloqués avec 429 error Too many request. 
6. La météo de Lyon change-t-elle toutes les secondes ? Qu'est-ce que ça vous suggère en plus ?
Non, la météo ne change pas toutes les secondes : une fois les données récupérées, on les garde dans un cache et on ne les met à jour qu'après un certain délai (par exemple 10 minutes), ce qui évite de payer le fournisseur à chaque requête.

## Cas 7 La mise à jour du 3 mars
1. Pour chacune des 5 lignes : cassante ou non cassante ? Justifiez en pensant à une appli installée il y a six mois.
- Ajout du champ currency dans GET /v1/accounts/{id} : Non cassante. Ajouter un nouveau champ dans la réponse n'empêche pas une appli installée il y a six mois de fonctionner (elle ignore simplement les champs qu'elle ne connaît pas).
- Renommage de balance en availableBalance : Cassante. L'appli installée il y a six mois attend balance dans le JSON ; si ce champ disparaît ou change de nom, l'affichage du solde échoue.
- Suppression du champ iban (peu utilisé) : Cassante. Même s'il est peu utilisé, il reste utilisé par certains écrans ou parcours ; le supprimer d'un coup casse ces fonctionnalités pour les clients concernés.
- Nouveau paramètre obligatoire reason sur POST /v1/transfers : Cassante. Les anciennes versions de l'appli n'envoient pas reason et recevront des erreurs de validation (400 / 422), bloquant les virements.
- Ajout de la route GET /v1/accounts/{id}/cards : Non cassante. C'est une nouvelle route qui n'altère pas le comportement des routes déjà en place.
On ajoute, on ne casse pas 
2. Quel dégât concret pour la banque ?
Blocage des opérations clients et saturation du support: 4000 appel au support de bank, impossibilité de consulter les information et de faire des virements pour certains clients avec app qui n'est pas mis à jour. Perte de confiance des clients. 
3. Quel principe du cours s'applique ?
On ajoute, on ne casse pas
4. Réécrivez le plan de mise à jour : qu'est-ce qui peut partir dans la version 1, qu'est-ce qui demande une version 2, et que devient la version 1 ?
Ce qui part dans la version 1 (v1, 100 % rétrocompatible) :
Ajout du champ currency dans GET /v1/accounts/{id}.
Ajout de la route GET /v1/accounts/{id}/cards.
Ajout du paramètre reason sur POST /v1/transfers en optionnel (avec une valeur par défaut côté serveur si l'ancienne appli ne le fournit pas).

Ce qui demande une version 2 (v2) :
Le renommage de balance en availableBalance, la suppression définitive de iban, et l'obligation stricte d'envoyer reason.

Que devient la version 1 ?
Elle reste active et maintenue pendant une période de transition le temps que les 200 000 téléphones migrent vers la nouvelle version de l'application, avec un avertissement de dépréciation avant son arrêt futur avec date limite. 

5. Comment obtenir le même résultat métier pour la ligne 2 sans rien casser ?
On conserve balance dans GET /v1/accounts/{id} et on ajoute availableBalance en parallèle avec la même valeur : les anciennes applis continuent de lire balance, tandis que les nouvelles utilisent availableBalance.

## Cas 8 La conversation
1. Où est le problème ? Décrivez-le en une phrase, comme à un collègue.
Il manque un contrat d'API écrit : sans documentation formelle, le frontend et le backend se sont mis d'accord à l'oral sur un format approximatif ({price, date}), ce qui crée des incohérences sur les noms de champs, les unités et les formats, et provoque des bugs d'affichage côté client.
2. Quel dégât concret : prix, sécurité ou scalabilité ? Donnez un exemple chiffré ou une situation.
un dégât de qualité de service / temps perdu (coût humain et bug métier critique)
Pendant 3 semaines, le front et le back développent en parallèle sur des bases incompatibles. À la mise en commun, tous les écrans affichent 2490 € au lieu de 24,90 € (un prix multiplié par 100 !) et Invalid Date, obligeant les deux équipes à tout reprendre en urgence avant la livraison.
3. Quel principe du cours s'applique ?
Le contrat d'API d'abord
4. Qu'aurait-il fallu produire le premier jour, avant d'écrire une ligne de code ? Décrivez ce document et ce qu'il contient pour cette route.
Document à produire : Un contrat d'API (spécification OpenAPI / Swagger, en YAML ou JSON) validé ensemble par le front et le back.
L'URL et la méthode (GET /orders/{id}).
Ce qu'on envoie (id dans l'URL).
Ce que le serveur renvoie (amount_cents en centimes, created_at en date ISO).
Les codes de réponse (200 si ça marche, 404 si la commande n'existe pas).
```
# Contrat d'API : GET /orders/{id}
route: GET /orders/{id}

parametre_url:
  id: integer (obligatoire, ex: 42)

reponses:
  200_OK:
    id: integer (ex: 42)
    amount_cents: integer # en centimes ! 2490 = 24,90 €
    created_at: string  # format ISO 8601, ex: "2026-11-20T10:00:00Z"

  404_Not_Found:
    error: string (ex: "Commande introuvable")
```
5. Relevez tous les malentendus de la conversation. Il y en a au moins quatre.
Accord flou au départ : Tom donne {price, date} en gros sans préciser noms, types ou formats, et Sarah code dessus.
Nom du prix : Sarah attend price, le back renvoie amount_cents.
Unité du prix : Sarah pense recevoir des euros (24.90), le back renvoie des centimes (2490 → affichage 2490 €).
Nom et format de la date : Sarah attend date, le back renvoie created en timestamp Unix (1795000000 → Invalid Date).
Changement mal communiqué : Tom a changé le nom lundi à l'oral avec Karim, sans prévenir Sarah ni mettre à jour un contrat.

## Cas 9 Le test qui dit toujours oui
1. Où est le problème ? Décrivez-le en une phrase, comme à un collègue.
Ce test vérifie uniquement le statut 200 sans contrôler le montant final ni les conditions d'application de la remise, ce qui produit un « faux vert » et masque le double calcul de la réduction en production.
2. Quel dégât concret : prix, sécurité ou scalabilité ? Donnez un exemple chiffré ou une situation.
- Le prix direct : si un panier fait 50 € avec -10 %, il doit passer à 45 €. Là, comme la remise passe deux fois (50 × 0,9 × 0,9), le client paie 40,50 € : tu perds 4,50 € à chaque commande (soit 450 € envolés sur 100 commandes).
- les faux verts font perdre confiance dans les tests au moment de déployer
3. Quel principe du cours s'applique ?
Ce qui n'est pas testé est déjà cassé
4. Réécrivez ce test pour qu'il attrape le bug.
```
    test('applique une seule fois -10 % sur un panier à 50 € (total 45 €)', async () => {
    const res = await api.post('/v1/carts/7/checkout', { promoCode: 'BIENVENUE10' });

    expect(res.status).toBe(200);
    expect(res.body.subtotal).toBe(50);
    expect(res.body.discountAmount).toBe(5);
    expect(res.body.total).toBe(45); // Tombe en échec si ça sort 40,50 €
    });
```
5. Citez deux autres cas qu'il faudrait tester sur cette route.
- Vérifier qu'un code déjà utilisé ou des conditions non respectées n'appliquent pas la remise (ou ne la doublent pas).
- Vérifier qu'un code faux ou expiré, ou l'accès par un mauvais utilisateur, renvoie bien une erreur.