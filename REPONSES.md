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
4. Réécrivez les cinq routes proprement, avec leur méthode HTTP.
   GET  /student
   DELETE /student/4
   GET /notes
   POST /note
   GET /student/note/4
5. La dernière ligne cache un danger particulier. Lequel ?
  - /getStudentNotes?student=4&delete=true expose ID de student et aussi expose params delete boolean.
**Notre correction :**
(les routes réécrites, la réponse HTTP corrigée, ou quelques lignes de code)

## Cas 2
...
