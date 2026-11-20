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


## Cas 3 La page d'accueil met 9 secondes
1. Il envoye tous les produits de 80000 juste pour affiche 12 items sur premier écran et aussi données data depuis DB sans DTO, qui expose tous les infos. 
2. Envoyer 80000 items dont 12 utilisés et cela peut augmenter les prix de db cloud(supebase), et baisser la performance, et aussi bloque scalabilité pour gérer plus de données avec plus de clients. 
- Envoyer tous les donnés de produits directement de DB sans DTO peux exposer trop d'infos, cela augmente les risques d'attack contre db. 
3. 3 Jamais de liste sans limite, 5 Ne envoyer pas que ce que l'ecran affiche
4. frontend
    ```
    GET /v1/products?limit=12&offset=0
    ```
    la réponse de l'API
    ```
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
    backend 
    ```
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
...
