# Études de cas — Groupe : ..............

## Cas 1

| Où est le problème ? | Quel dégât ? (prix, sécurité, scalabilité) | Quel principe du cours ? |
|----------------------|---------------------------------------------|--------------------------|
| Je trouve que les chemins a de problème comme nommage irregulier(camelcase,snakecase),certains routes inclutent les verbes,delete par POST, GET methode sans utiliser DELETE methode, et manque d'interface uniforme. | **Scalabilité et prix** : `GET /students` ou `/notes_list` sans pagination renvoie les 5 000 éléments à chaque appel. À environ 1 Ko par élément, cela fait 5 Mo par appel ; avec 1 000 appels par jour, on transfère 5 Go par jour pour rien. La facture de la base cloud (Supabase) augmente, les temps de réponse aussi, et la situation empire à chaque nouvel étudiant.<br>**Sécurité** : une suppression déclenchée par un GET peut être lancée par accident (voir question 5).<br>**Compatibilité** : si on corrige les noms des routes du jour au lendemain, toutes les applis qui utilisent déjà l'API cassent. Il faut garder les anciennes routes le temps de la migration, ou publier les nouvelles sous `/v2`. | 1 Des noms dans l'URL, l'action dans le verbe<br>2 Le code HTTP dit la vérité(Utilise GET/POST pour DELETE envoie 200 mais cela pas correspond pas la vérité?)<br>3 Jamais de liste sans limite<br>On ajoute, on ne casse pas( Pour mettre à jour de API) |

**Notre correction :**

```http
GET    /students
DELETE /students/4
GET    /notes
POST   /notes
GET    /students/4/notes
```

**Questions complémentaires :**

**5. La dernière ligne cache un danger particulier. Lequel ?**

/getStudentNotes?student=4&delete=true expose ID de student et aussi expose params delete boolean et logique delete interieur.

## Cas 2 — La place F7

| Où est le problème ? | Quel dégât ? (prix, sécurité, scalabilité) | Quel principe du cours ? |
|----------------------|---------------------------------------------|--------------------------|
| Il envoie 200 avec un message d'erreur pour POST booking, mais il faut plutôt envoyer 409 Conflict, parce que le code HTTP dit la vérité en tant que convention et pour éviter les contradictions dans l'application, et toujours refléter les vrais états réels. | L'utilisateur croit que sa réservation a réussi parce que l'app reçoit 200 et affiche « réussi » sans message d'erreur. Résultat : ce n'est pas le cas, il n'a pas pu réserver, ce qui coûte une baisse du chiffre d'affaires et une perte de clients au cinéma. Les spectateurs arrivent dans la salle et il y a une queue, parce qu'une personne ne peut pas trouver sa place à cause d'un double booking sur le site. | Le code HTTP dit la vérité |

**Notre correction :**

```http
HTTP/1.1 409 Conflict
Content-Type: application/json

{ "movieId": 12, "seat": "F7" , "message": "Already reserved this seat."}
```

**Questions complémentaires :**

**5.**

Un conflit de réplication (souvent associé à un code HTTP 409 ou une erreur applicative E42) se produit lorsque deux utilisateurs ou appareils modifient le même enregistrement hors ligne avant de se synchroniser.
C'est que le developpeur d'app qui doit savoir.

## Cas 3 — La page d'accueil met 9 secondes

| Où est le problème ? | Quel dégât ? (prix, sécurité, scalabilité) | Quel principe du cours ? |
|----------------------|---------------------------------------------|--------------------------|
| Il envoie tous les 80 000 produits juste pour afficher 12 items sur le premier écran, et envoie aussi les données depuis la DB sans DTO, ce qui expose toutes les infos. | Envoyer 80 000 items dont 12 utilisés peut augmenter le prix de la DB cloud (Supabase), baisser la performance, et aussi bloquer la scalabilité pour gérer plus de données avec plus de clients.<br>Envoyer toutes les données des produits directement depuis la DB sans DTO peut exposer trop d'infos, cela augmente les risques d'attaque contre la DB. | **Principe 3** : Jamais de liste sans limite<br>**Principe 5** : N'envoyer que ce que l'écran affiche |

**Notre correction :**

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

**Questions complémentaires :**

**5. Le front demande 500 000 produits d'un coup. Que fait votre API ?**

Envoyer BAD Request 400, 422

## Cas 4 — Trois dons de 20 €

| Où est le problème ? | Quel dégât ? (prix, sécurité, scalabilité) | Quel principe du cours ? |
|----------------------|---------------------------------------------|--------------------------|
| L'endpoint POST /v1/donations n'est pas idempotent (pas de clé d'idempotence ni de vérification de doublon) et déclenche un débit bancaire à chaque tentative, ce qui fait que les réessais réseau de l'appli créent plusieurs dons et plusieurs débits de 20 €. | Les payments inattendu pour l'utilisateur, il faut remboursement, risque de perte de client, basse de qualité de service. | Un compteur à l'entrée |

**Notre correction :**

*Solution étape par étape :*

* **Ce que fait l'appli :** elle crée une clé unique (clé d'idempotence) pour ce don et l'envoie avec la requête. En cas de coupure réseau et de réessai, elle **garde exactement la même clé**.
* **Ce que fait le serveur au 1er envoi :** il enregistre la clé comme « en cours », crée le don, fait le débit bancaire, sauvegarde le résultat, puis renvoie **`201`** avec le don.
* **Ce que fait le serveur au 2e envoi (avec la même clé) :**
  * Si c'est **déjà terminé**, il **ne redébite pas** et renvoie directement le résultat déjà enregistré (`201`).
  * Si c'est **encore en cours**, il **ne redébite pas** non plus et indique que le traitement est déjà en cours.

```js
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

**Questions complémentaires :**

**5. Regardez l'ordre des deux opérations dans le code. Que se passe-t-il si le débit échoue ?**

Avec le premier code, il ne fait rien. Le code proposé par 4, on va mettre à jour de l'état de clé d'idempotent.

## Cas 5 — La fiche du conducteur

| Où est le problème ? | Quel dégât ? (prix, sécurité, scalabilité) | Quel principe du cours ? |
|----------------------|---------------------------------------------|--------------------------|
| Il envoie les informations privées importantes comme l'IBAN, le password. | Cela amène une grande fuite d'informations de clients et une mauvaise utilisation. Il faut compenser les victimes, et aussi baisse de crédibilité du service, et perte de clients. | N'envoie pas ce que l'écran affiche |

**Notre correction :**

*Classement des 8 champs :*

| Visible par tous | Réservé au conducteur lui-même | Ne sort jamais de l'API |
|------------------|--------------------------------|-------------------------|
| `id`             | `phone`                        | `passwordHash`          |
| `firstName`      | `homeAddress`                  | `iban`                  |
| `rating`         | `licenseNumber`                |                         |

**Questions complémentaires :**

**5. « Ce n'est pas grave, le front ne les affiche pas. » Que répondez-vous ?**

Dès l'instant où une donnée est incluse dans la réponse JSON, elle est déjà partie sur le réseau : même si l'interface ne l'affiche pas, n'importe qui peut facilement la voir via les outils de développement du navigateur (onglet Réseau), curl ou Postman.

## Cas 6 — La facture de mars

| Où est le problème ? | Quel dégât ? (prix, sécurité, scalabilité) | Quel principe du cours ? |
|----------------------|---------------------------------------------|--------------------------|
| L'endpoint GET /v1/forecast est ouvert sans authentification ni limitation, n'importe quoi peut acceder et appelle un fournisseur externe payant à chaque requête sans mise en cache, ce qui provoque une explosion des appels quand il y a trop d'access. | Catégorie principale : Prix (et scalabilité / dépendance externe).<br>Exemple chiffré : à 0,002 € par appel :<br>10 000 appels/jour = 20 €/jour (600 €/mois).<br>Multiplié par 40 (400 000 appels/jour) = 800 €/jour (24 000 €/mois) rien que pour un seul endpoint, sans compter le risque d'atteindre les quotas ou latences du fournisseur externe. | **Un compteur à l'entrée**, parce que chaque client doit avoir une limite, et qu'au-delà l'API doit répondre `429` **avant** d'appeler la base ou le fournisseur payant. |

**Notre correction :**

- **Qui est compté :** l'adresse IP du client (ou une clé d'API si on identifie les partenaires).
- **Quelle limite :** par exemple 60 requêtes par minute par adresse IP (un seuil cohérent avec un usage normal, mais qui bloque le polling à 1 requête/seconde).
- **Ce que l'API répond quand la limite est dépassée :**
  - Le code HTTP 429 Too Many Requests.
  - L'en-tête Retry-After (ex. Retry-After: 60) indiquant dans combien de secondes réessayer.
  - Un message JSON clair expliquant que le quota est dépassé, sans appeler le fournisseur (donc 0 € facturé).

**Questions complémentaires :**

**5. Vous limitez par adresse IP. Une école de 800 étudiants sort sur Internet avec une seule adresse. Que se passe-t-il ?**

Ils sont consideré de meme adress et seront bloqués avec 429 error Too many request.

**6. La météo de Lyon change-t-elle toutes les secondes ? Qu'est-ce que ça vous suggère en plus ?**

Non, la météo ne change pas toutes les secondes : une fois les données récupérées, on les garde dans un cache et on ne les met à jour qu'après un certain délai (par exemple 10 minutes), ce qui évite de payer le fournisseur à chaque requête.

## Cas 7 — La mise à jour du 3 mars

| Où est le problème ? | Quel dégât ? (prix, sécurité, scalabilité) | Quel principe du cours ? |
|----------------------|---------------------------------------------|--------------------------|
| 3 changements cassants ont été publiés dans la v1, alors que les 200 000 téléphones qui n'ont pas mis à jour l'appli utilisent encore l'ancien format. | Blocage des opérations clients et saturation du support: 4000 appel au support de bank, impossibilité de consulter les information et de faire des virements pour certains clients avec app qui n'est pas mis à jour. Perte de confiance des clients. | On ajoute, on ne casse pas |

**Notre correction :**

*Plan de mise à jour :*

- **Ce qui part dans la version 1 (v1, 100 % rétrocompatible) :**
  - Ajout du champ currency dans GET /v1/accounts/{id}.
  - Ajout de la route GET /v1/accounts/{id}/cards.
  - Ajout du paramètre reason sur POST /v1/transfers en optionnel (avec une valeur par défaut côté serveur si l'ancienne appli ne le fournit pas).
- **Ce qui demande une version 2 (v2) :** Le renommage de balance en availableBalance, la suppression définitive de iban, et l'obligation stricte d'envoyer reason.
- **Que devient la version 1 ?** Elle reste active et maintenue pendant une période de transition le temps que les 200 000 téléphones migrent vers la nouvelle version de l'application, avec un avertissement de dépréciation avant son arrêt futur avec date limite.

*Même résultat métier pour la ligne 2 sans rien casser :*

On conserve balance dans GET /v1/accounts/{id} et on ajoute availableBalance en parallèle avec la même valeur : les anciennes applis continuent de lire balance, tandis que les nouvelles utilisent availableBalance.

**Questions complémentaires :**

**1. Pour chacune des 5 lignes : cassante ou non cassante ?**

| Changement | Cassante ? | Justification (appli installée il y a six mois) |
|------------|------------|-------------------------------------------------|
| Ajout du champ currency dans GET /v1/accounts/{id} | Non cassante | Ajouter un nouveau champ dans la réponse n'empêche pas une appli installée il y a six mois de fonctionner (elle ignore simplement les champs qu'elle ne connaît pas). |
| Renommage de balance en availableBalance | Cassante | L'appli installée il y a six mois attend balance dans le JSON ; si ce champ disparaît ou change de nom, l'affichage du solde échoue. |
| Suppression du champ iban (peu utilisé) | Cassante | Même s'il est peu utilisé, il reste utilisé par certains écrans ou parcours ; le supprimer d'un coup casse ces fonctionnalités pour les clients concernés. |
| Nouveau paramètre obligatoire reason sur POST /v1/transfers | Cassante | Les anciennes versions de l'appli n'envoient pas reason et recevront des erreurs de validation (400 / 422), bloquant les virements. |
| Ajout de la route GET /v1/accounts/{id}/cards | Non cassante | C'est une nouvelle route qui n'altère pas le comportement des routes déjà en place. |

## Cas 8 — La conversation

| Où est le problème ? | Quel dégât ? (prix, sécurité, scalabilité) | Quel principe du cours ? |
|----------------------|---------------------------------------------|--------------------------|
| Il manque un contrat d'API écrit : sans documentation formelle, le frontend et le backend se sont mis d'accord à l'oral sur un format approximatif ({price, date}), ce qui crée des incohérences sur les noms de champs, les unités et les formats, et provoque des bugs d'affichage côté client. | un dégât de qualité de service / temps perdu (coût humain et bug métier critique)<br>Pendant 3 semaines, le front et le back développent en parallèle sur des bases incompatibles. À la mise en commun, tous les écrans affichent 2490 € au lieu de 24,90 € (un prix multiplié par 100 !) et Invalid Date, obligeant les deux équipes à tout reprendre en urgence avant la livraison. | Le contrat d'API d'abord |

**Notre correction :**

Document à produire : Un contrat d'API (spécification OpenAPI / Swagger, en YAML ou JSON) validé ensemble par le front et le back.

- L'URL et la méthode (GET /orders/{id}).
- Ce qu'on envoie (id dans l'URL).
- Ce que le serveur renvoie (amount_cents en centimes, created_at en date ISO).
- Les codes de réponse (200 si ça marche, 404 si la commande n'existe pas).

```yaml
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

**Questions complémentaires :**

**5. Relevez tous les malentendus de la conversation. Il y en a au moins quatre.**

1. Accord flou au départ : Tom donne {price, date} en gros sans préciser noms, types ou formats, et Sarah code dessus.
2. Nom du prix : Sarah attend price, le back renvoie amount_cents.
3. Unité du prix : Sarah pense recevoir des euros (24.90), le back renvoie des centimes (2490 → affichage 2490 €).
4. Nom et format de la date : Sarah attend date, le back renvoie created en timestamp Unix (1795000000 → Invalid Date).
5. Changement mal communiqué : Tom a changé le nom lundi à l'oral avec Karim, sans prévenir Sarah ni mettre à jour un contrat.

## Cas 9 — Le test qui dit toujours oui

| Où est le problème ? | Quel dégât ? (prix, sécurité, scalabilité) | Quel principe du cours ? |
|----------------------|---------------------------------------------|--------------------------|
| Ce test vérifie uniquement le statut 200 sans contrôler le montant final ni les conditions d'application de la remise, ce qui produit un « faux vert » et masque le double calcul de la réduction en production. | Le prix direct : si un panier fait 50 € avec -10 %, il doit passer à 45 €. Là, comme la remise passe deux fois (50 × 0,9 × 0,9), le client paie 40,50 € : tu perds 4,50 € à chaque commande (soit 450 € envolés sur 100 commandes).<br>les faux verts font perdre confiance dans les tests au moment de déployer | Ce qui n'est pas testé est déjà cassé |

**Notre correction :**

```js
test('applique une seule fois -10 % sur un panier à 50 € (total 45 €)', async () => {
  const res = await api.post('/v1/carts/7/checkout', { promoCode: 'BIENVENUE10' });

  expect(res.status).toBe(200);
  expect(res.body.subtotal).toBe(50);
  expect(res.body.discountAmount).toBe(5);
  expect(res.body.total).toBe(45); // Tombe en échec si ça sort 40,50 €
});
```

**Questions complémentaires :**

**5. Citez deux autres cas qu'il faudrait tester sur cette route.**

- Vérifier qu'un code déjà utilisé ou des conditions non respectées n'appliquent pas la remise (ou ne la doublent pas).
- Vérifier qu'un code faux ou expiré, ou l'accès par un mauvais utilisateur, renvoie bien une erreur.

## Cas 10 — Sept lignes, combien de problèmes ?

**1. Listez tous les problèmes que vous trouvez. Pour chacun : le principe du cours concerné.**

| Où est le problème ? | Détail | Quel principe du cours ? |
|----------------------|--------|--------------------------|
| IDOR / pas d'autorisation | n'importe qui lit n'importe quel client | Sécurité / contrôle d'accès. |
| Pas de validation d'entrée | req.query.id non vérifié (400) | Validation des entrées. |
| Pas de 404 / crash si null | user.id plante si introuvable | Gestion d'erreur. |
| Fuite de champs user | risque d'exposer hash/données privées | Moindre privilège / filtrage de sortie. |
| Pas de pagination sur orders | 3 000 commandes chargées d'un coup | Scalabilité / performance. |
| Pas de try/catch + URL avec verbe | erreur brute / /getUserOrders non REST | Fiabilité & design d'API. |

**2. Que se passe-t-il si l'identifiant demandé n'existe pas ? Suivez le code ligne par ligne.**

| Ligne | Ce qui se passe |
|-------|-----------------|
| 1 | requête acceptée sans filtre. |
| 2 | findById renvoie null (ou undefined). |
| 3 | user.id lève un TypeError (promesse rejetée → 500 ou crash). |
| 4 | jamais exécutée. |

**3. Classez vos problèmes du plus grave au moins grave, et défendez votre classement.**

1. **IDOR / accès aux données d'autrui** : fuite directe de données clients.
2. **Fuite de champs sensibles (user)** : exposition de données privées/secrets.
3. **Absence de pagination (orders)** : surcharge mémoire/latence dès qu'il y a beaucoup de commandes.
4. **Absence de 404 et de try/catch** : fausse erreur technique et risque de crash.
5. **Absence de validation (400)** : entrées sales envoyées à la DB.
6. **URL avec verbe (/getUserOrders)** : moins critique en prod, mais mauvais design/maintenabilité.

**Notre correction :**

*4. La route réécrite :*

```js
app.get('/v1/users/:userId/orders', requireAuth, async (req, res, next) => {
  try {
    const { userId } = req.params;
    if (!isValidId(userId)) {
      return res.status(400).json({ error: { code: 'INVALID_ID', message: 'ID invalide.' } });
    }
    if (!canReadOrders(req.user, userId)) {
      return res.status(403).json({ error: { code: 'FORBIDDEN', message: 'Accès interdit.' } });
    }
    const user = await db.users.findPublicById(userId);
    if (!user) {
      return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Introuvable.' } });
    }
    const limit = Math.min(Number(req.query.limit) || 20, 100);
    const orders = await db.orders.findPage({ userId, limit, cursor: req.query.cursor });
    return res.status(200).json({ user, orders });
  } catch (err) {
    return next(err);
  }
});
```
