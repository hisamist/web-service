# Études de cas — Groupe : ..............

## Cas 1
1. Où est le problème ? Décrivez-le en une phrase, comme à un collègue.
- Je trouve que les chemins a de problème comme nommage irregulier(camelcase,snakecase),certains routes inclutent les verbes,delete par POST, GET methode sans utiliser DELETE methode, et manque d'interface uniforme. 
2. Quel dégât concret : prix, sécurité ou scalabilité ? Donnez un exemple chiffré ou une situation.
- GET /students,/notes_list sans pagination peut envoyer 5000 items et cela peut augmenter les prix de db cloud(supebase), et baisser la performance, et aussi bloque scalabilité pour gérer plus de données avec plus de clients. 
- Pour mis à jour de nommages, cele peut empecher les consomateurs de cet API. 
3. Quel principe du cours s'applique ?
- 1 Des noms dans l'URL, l'action dans le verbe
- 2 Le code HTTP dit la vérité(Utilise GET/POST pour DELETE envoie 200 mais cela pas correspond pas la vérité?)
- 3 Jamais de liste sans limite 
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

## Cas 5

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
