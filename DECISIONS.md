# Journal de décisions — Groupe : ..............

Langage et framework choisis : JavaScript, ElysiaJS (sur le runtime Bun)
Base de données choisie : SQLite (via `bun:sqlite`, intégré à Bun)

## Pourquoi cette stack pour une API de cette taille ?

**ElysiaJS**, parce qu'il est rapide, léger, et qu'il intègre déjà les fonctionnalités dont le TP a besoin :

| Raison | Ce que ça apporte au TP |
|--------|-------------------------|
| **Validation intégrée** (schémas `t.Object`, `t.Number`…) sur le corps, les paramètres et la query | Un panier vide ou un `id` invalide est rejeté avant d'atteindre notre code : c'est la base de l'étape 2. |
| **Hooks de cycle de vie** (`onRequest`, `beforeHandle`, `onError`) | `onRequest` passe avant la lecture du corps et l'accès à la base : on y place le rate limiting (étape 6). `onError` centralise le format unique des erreurs (étape 2). |
| **Plugin OpenAPI / Swagger officiel** | Les schémas de validation génèrent une documentation qu'on peut comparer à notre contrat `openapi.yaml`. |
| **Léger** : peu de dépendances, pas de couche de compilation | Moins de code à comprendre et à expliquer à la démo. |
| **Rapide** : tourne sur Bun, conçu pour la performance | Démarrage instantané, et une boucle de 200 requêtes (étape 6) s'exécute en quelques secondes. |
| **Même langage que nos analyses** de la partie 1 (JavaScript) | On réutilise ce qu'on a déjà écrit (DTO, idempotence, pagination). |

**Bun**, parce qu'il fournit tout le reste sans outil supplémentaire :
- **SQLite intégré** (`bun:sqlite`) : pas de module natif à compiler sous Windows.
- **Lanceur de tests intégré** (`bun test`) : les tests se lancent en une seule commande (étape 8).

**SQLite**, parce que la base tient dans un seul fichier : rien à installer ni à héberger, et c'est largement suffisant pour 20 restaurants et 5 000 commandes.

**Limites qu'on assume :**
- Elysia et Bun sont plus récents qu'Express et Node : la communauté est plus petite, et on trouve moins d'exemples en ligne.
- SQLite gère mal beaucoup d'écritures simultanées, et le fichier ne se partage pas entre plusieurs serveurs. Ce n'est pas un problème pour le TP, mais ça compte pour la question des 3 serveurs (étape 6).

## Ressources et routes pour les 5 besoins

4 ressources : `restaurants`, `orders`, `payments`, `users`. Toutes les routes sont sous `/v1`, sans verbe dans l'URL. Le détail complet est dans [`openapi.yaml`](openapi.yaml).

| Besoin | Route | Quand tout va bien | Quand ça se passe mal |
|--------|-------|--------------------|-----------------------|
| Lister les commandes | `GET /v1/orders?limit=&cursor=` | `200` : une page de commandes (20 par défaut, 100 au maximum) et un `nextCursor` | `400` limit ou cursor invalide · `429` trop de requêtes |
| Consulter une commande | `GET /v1/orders/{orderId}` | `200` : la commande avec ses plats | `400` id invalide · `404` commande inexistante · `429` |
| Créer une commande | `POST /v1/restaurants/{restaurantId}/orders` | `201` : la commande créée, avec l'en-tête `Location` | `400` JSON illisible · `404` restaurant, utilisateur ou plat inexistant · `422` panier vide ou quantité invalide · `429` |
| Payer une commande | `POST /v1/orders/{orderId}/payments` + en-tête `Idempotency-Key` | `201` : le paiement ; même clé renvoyée = même réponse, aucun nouveau débit | `400` clé absente ou JSON illisible · `404` commande inexistante · `409` déjà payée · `422` clé réutilisée pour une autre requête · `429` |
| Consulter un profil | `GET /v1/users/{userId}` | `200` : `id`, `firstName`, `createdAt` uniquement | `400` id invalide · `404` utilisateur inexistant · `429` |

Toutes les erreurs ont la même forme :

```json
{ "error": { "code": "ORDER_NOT_FOUND", "message": "La commande 999999 n'existe pas." } }
```

## Journal par étape

| Étape | Ce qu'on a décidé | Pourquoi | Ce qu'on a écarté |
|-------|-------------------|----------|-------------------|
| 0 | Stack : JS + ElysiaJS + SQLite. Contrat OpenAPI 3.1 (`openapi.yaml`) commité avant le code. 4 ressources : `restaurants`, `orders`, `payments`, `users`, 5 routes sous `/v1`. Prix en **centimes entiers + devise** (`totalCents`, `currency`), dates en **ISO 8601 UTC**. Une forme d'erreur unique `{ error: { code, message, details? } }`. | Pour 5 routes, un seul serveur et une base dans un fichier suffisent : rien à installer ni à héberger. Les centimes évitent les erreurs d'arrondi et l'ambiguïté euros/centimes (cas 8). L'ISO 8601 évite « Invalid Date » et les fuseaux ambigus. Un `code` stable permet au client de réagir sans lire le message. | Express (mature, mais validation et doc à ajouter avec des bibliothèques en plus). PostgreSQL / Supabase (un serveur à installer, inutile à cette taille). Microservices, Docker, Kubernetes (la Lamborghini). Prix décimaux en euros (arrondis flottants). Timestamps Unix (illisibles, piège du cas 8). |
| 1 | | | |
| 2 | | | |
| 3 | *(proposé à l'étape 0, à confirmer)* Pagination par **curseur** : défaut 20, maximum 100, une `limit` plus grande est **plafonnée**. | Un curseur reste rapide quel que soit le volume (5 000 ou 5 millions), alors qu'`OFFSET` relit toutes les lignes sautées. | Numéro de page / `OFFSET`. Refuser une `limit` trop grande (plafonner est plus simple pour le client). |
| 4 | *(proposé à l'étape 0, à confirmer)* Clé d'idempotence générée par le **client**, dans l'en-tête `Idempotency-Key`, obligatoire. | | |
