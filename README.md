# TP3_Rest

TP3 REST : Microservices « Calculatrice » avec Spring Boot

## Compte-Rendu : TP3 - Microservices avec Spring Boot

### Contexte Universitaire

- Établissement : Faculté des Sciences de Sfax (FSS)
- Matière : Architecture MicroServices
- Filière : Master Professionnel (MPSRCC1)
- Année Universitaire : 2026-2027
- Enseignantes : Corinne Amel ZAYANI, Amal ABID
- Étudiant(e) : **Nour Elhak Ben Mefteh**

### Environnement

- JDK 17+
- Maven (intégré à IntelliJ)
- Spring Boot 4.0.9
- IntelliJ IDEA
- Postman et curl

### Structure du dépôt

```
TP3_Rest
├── calcul-service        (port 8081, ressource /calcul)
├── historique-service    (port 8082, ressource /operations)
└── README.md
```

---

## Partie A : Questions de conception

**1. Avantage et inconvénient d'un calcul par requête HTTP au lieu d'un appel de méthode.**

- Avantage : `calcul-service` est indépendant. On peut le modifier, le redéployer ou le dupliquer sans toucher à `historique-service`.
- Inconvénient : l'appel passe par le réseau. Il est plus lent qu'un appel de méthode, et `historique-service` dépend de la disponibilité de `calcul-service`.

**2. Si `calcul-service` est arrêté :**

- Continuent de fonctionner : `GET /operations`, `GET /operations/{id}` et `DELETE /operations/{id}`. Ces requêtes n'utilisent que les données en mémoire d'`historique-service`.
- Échouent : les quatre `GET :8081/calcul/...` (le service lui-même est arrêté), `POST /operations` et `PUT /operations/{id}` (ils ont besoin du calcul).

**3. Pourquoi deux microservices ne peuvent pas écouter sur le même port ?**
Un port ne peut être ouvert que par un seul processus à la fois. Un second programme sur le même port échoue avec l'erreur `Port 8081 was already in use`.

**4. Code de statut quand `calcul-service` est injoignable :**
**503 Service Unavailable**. Le client n'a pas fait d'erreur et le serveur n'a pas de bug : le service dont on dépend est indisponible.

---

## Partie C : Premier microservice

### Correspondance Spring / JAX-RS (C.2)

| Spring (TP3) | JAX-RS (TP2) | Rôle dans cet exemple |
|---|---|---|
| `@RestController` | (pas d'équivalent direct) | Déclare la classe comme contrôleur REST : les valeurs retournées sont écrites dans le corps de la réponse |
| `@RequestMapping("/calcul")` | `@Path("/calcul")` sur la classe | Chemin de base de toutes les méthodes de la classe |
| `@GetMapping("/somme")` | `@GET` + `@Path("/somme")` | La méthode répond aux requêtes GET sur `/calcul/somme` |
| `produces = MediaType.TEXT_PLAIN_VALUE` | `@Produces(MediaType.TEXT_PLAIN)` | La réponse est du texte brut |
| `@RequestParam("a") int a` | `@QueryParam("a") int a` | Lit le paramètre `a` de l'URL (`?a=2`) |
| `ResponseEntity.ok(...)` | `Response.ok(...).build()` | Construit une réponse de statut 200 avec un corps |

### Comprendre l'URL (C.6)

URL : `http://localhost:8081/calcul/somme?a=2&b=3`

| Partie de l'URL | Origine |
|---|---|
| `localhost:8081` | Machine locale et port fixé par `server.port` dans `application.properties` |
| `/calcul` | Annotation `@RequestMapping("/calcul")` de la classe |
| `/somme` | Annotation `@GetMapping("/somme")` de la méthode |
| `?a=2&b=3` | Paramètres de requête lus par `@RequestParam("a")` et `@RequestParam("b")` |

**Question C.1. Quelles parties de l'URL du TP2 ont disparu, et pourquoi ?**
Ont disparu `/RestWebApp` et `/api`. `/RestWebApp` était le nom du fichier `.war` déployé dans Tomcat (le contexte de l'application). `/api` venait de `@ApplicationPath("/api")`. Avec Spring Boot, il n'y a plus de `.war` ni de Tomcat externe : l'application a son propre serveur embarqué, avec le contexte `/`. Le port aussi a changé (8080 → 8081), choisi dans `application.properties`.

**Question C.2. Que donne un POST sur `/calcul/somme` ? Est-ce le même statut qu'au TP2 ?**
On obtient **405 Method Not Allowed** : la méthode n'accepte que GET. C'est le même statut qu'au TP2.

---

## Partie D : Compléter la calculatrice

### Observations (D.3)

| Requête | Statut | Corps |
|---|---|---|
| `GET /calcul/somme?a=2` (`b` absent) | 400 | JSON d'erreur Spring |
| `GET /calcul/somme?a=abc&b=3` (`a` non entier) | 400 | JSON d'erreur Spring |

**Question D.1. Que fait Spring pour un paramètre absent ? Quelle attitude est la plus conforme à REST ?**
Spring refuse la requête avec **400 Bad Request** et n'exécute pas la méthode (même réaction si le paramètre n'est pas un entier). JAX-RS, lui, donnait 0 au paramètre absent. L'attitude de Spring est plus conforme à REST : une requête incorrecte est signalée par un code 4xx, au lieu de renvoyer un résultat faux sans prévenir.

**Question D.2. Quels champs contient l'objet JSON d'erreur de Spring ?**
Quatre champs : `timestamp`, `status`, `error` et `path`.

**Question D.3. Correspondance des annotations.**

| Spring | JAX-RS |
|---|---|
| `@RequestParam` | `@QueryParam` |
| `@PathVariable` | `@PathParam` |

Pour lire l'identifiant dans `/operations/{id}`, on utilise **`@PathVariable`**.

---

## Partie E : Échanger des données en JSON

**Question E.1. Les deux textes JSON sont-ils équivalents ? Qui a fait la conversion ?**
Oui. Un objet JSON est un ensemble de paires clé/valeur sans ordre : le client lit les champs par leur nom, donc l'ordre n'a aucune importance. Cette fois, la conversion a été faite par **Jackson** (inclus dans Spring Boot), à la place de JSON-B.

**Question E.2. Pourquoi `Accept: application/xml` est-il refusé (406) alors que le code ne mentionne aucun format ?**
Spring choisit le format de la réponse d'après l'en-tête `Accept` et les convertisseurs disponibles. Pour un objet `Resultat`, seul Jackson (JSON) est présent dans le projet : aucun convertisseur XML. Le serveur ne peut donc pas produire du XML et répond **406 Not Acceptable**.

---

## Partie F : historique-service

### Tests F.6 : création et lecture

| N° | Requête | Corps envoyé | Statut | Réponse |
|---|---|---|---|---|
| 1 | POST /operations | `{"operateur":"somme","a":8,"b":2}` | 201 | `{"id":1,"operateur":"somme","a":8,"b":2,"resultat":10.0}` + en-tête `Location` |
| 2 | POST /operations | `{"operateur":"division","a":7,"b":2}` | 201 | `{"id":2,"operateur":"division","a":7,"b":2,"resultat":3.5}` |
| 3 | GET /operations | Aucun | 200 | Liste des 2 opérations |
| 4 | GET /operations/2 | Aucun | 200 | L'opération n° 2 (division 7 / 2 = 3.5) |
| 5 | GET /operations/99 | Aucun | 404 | Aucun corps |
| 6 | POST /operations | `{"operateur":"division","a":7,"b":0}` | 400 | `{"erreur":"Division par zero impossible"}` |
| 7 | POST /operations | `{"operateur":"racine","a":9,"b":0}` | 400 | `{"erreur":"Operateur invalide"}` |

### Tests F.7 : `calcul-service` arrêté

| N° | Requête | Corps envoyé | Statut | Réponse |
|---|---|---|---|---|
| 8 | GET /operations | Aucun | 200 | Liste des opérations déjà enregistrées |
| 9 | GET /operations/1 | Aucun | 200 | L'opération n° 1 |
| 10 | POST /operations | `{"operateur":"produit","a":3,"b":4}` | 503 | `{"erreur":"..."}` : `calcul-service` indisponible |

Après redémarrage de `calcul-service`, la requête 10 répond **201** avec l'identifiant 3.

**Question F.1. Pourquoi les lectures continuent-elles de fonctionner ? Était-ce possible au TP2 ?**
Les lectures ne lisent que la mémoire d'`historique-service` et n'appellent jamais `calcul-service`. Au TP2, ce n'était pas possible : tout était dans la même application `RestWebApp.war`, et le calcul était un simple appel de méthode. Si l'application s'arrêtait, tout s'arrêtait. Il n'y avait pas de panne partielle.

**Question F.2. Que se passe-t-il sans le bloc `catch (ResourceAccessException e)` ?**
L'exception n'est pas interceptée et remonte jusqu'à Spring, qui répond **500 Internal Server Error** avec le JSON d'erreur standard. Le client croit à un bug du serveur, alors que le vrai problème est l'indisponibilité de `calcul-service`. Le `catch` permet de renvoyer le code approprié, 503.

### F.8 : PUT /operations/{id}

| Situation | Statut | Corps de la réponse |
|---|---|---|
| Aucune opération ne porte cet identifiant | 404 | Aucun corps |
| Le calcul est impossible (opérateur inconnu, division par 0) | 400 | `{"erreur":"..."}` |
| `calcul-service` est injoignable | 503 | `{"erreur":"..."}` |
| Modification réussie | 200 | L'opération modifiée (ex. : `resultat` = 16.0) |

Test : `PUT /operations/1` avec `{"operateur":"produit","a":8,"b":2}` répond 200 avec `resultat` 16.0, puis `GET /operations/1` confirme la modification.

### F.9 : DELETE /operations/{id}

| Situation | Statut | Corps de la réponse |
|---|---|---|
| Aucune opération ne porte cet identifiant | 404 | Aucun corps |
| Suppression réussie | 204 | Aucun corps |

Tests : `DELETE /operations/1` → **204**, puis `GET /operations/1` → **404**, puis `DELETE /operations/1` → **404**.

---

## Questions de synthèse

**1. Tableau A.3 : étape où la différence est observée**

| Aspect | Étape |
|---|---|
| Technologie (Spring Boot au lieu de JAX-RS) | B.1, B.3 |
| Découpage en deux applications | B.1, F.1 |
| Résultat de la construction (`.jar`) | B.3, C.5 |
| Serveur (Tomcat embarqué) | C.5 (ligne `Tomcat started on port 8081`) |
| Port (un par microservice) | C.4, F.1 |
| Adresse (sans `/RestWebApp/api`) | C.6, C.7 |
| Activation (`@SpringBootApplication`) | B.4 |
| Ressource REST (`@RestController`) | C.2 |
| Verbes HTTP | C.2, F.5, F.8, F.9 |
| Paramètres | C.2, D.4, F.5 |
| Réponse HTTP (`ResponseEntity`) | C.2, D.2 |
| Conversion JSON (Jackson) | E.2 |
| Création des objets (`@Service`, singleton) | C.1, F.4 |
| Communication (requête HTTP) | F.3, F.6 |

**2. Correspondance complète des annotations JAX-RS / Spring**

| JAX-RS (TP2) | Spring (TP3) |
|---|---|
| `@ApplicationPath("/api")` + classe `Application` | `@SpringBootApplication` + méthode `main` |
| `@Path("/calcul")` (classe) | `@RestController` + `@RequestMapping("/calcul")` |
| `@Path("/somme")` (méthode) | Chemin dans `@GetMapping("/somme")` |
| `@GET` | `@GetMapping` |
| `@POST` | `@PostMapping` |
| `@PUT` | `@PutMapping` |
| `@DELETE` | `@DeleteMapping` |
| `@QueryParam` | `@RequestParam` |
| `@PathParam` | `@PathVariable` |
| `@Consumes(APPLICATION_JSON)` + paramètre | `@RequestBody` |
| `@Produces(...)` | `produces = ...` |
| `Response` | `ResponseEntity` |
| `uriInfo.getAbsolutePathBuilder()` | `ServletUriComponentsBuilder.fromCurrentRequest()` |
| JSON-B | Jackson |

**3. Construire et lancer les deux microservices sans IntelliJ**

Terminal 1 :

```
cd calcul-service
mvn clean package
java -jar target/calcul-service-0.0.1-SNAPSHOT.jar
```

Terminal 2 :

```
cd historique-service
mvn clean package
java -jar target/historique-service-0.0.1-SNAPSHOT.jar
```

Il n'y a rien à déployer : chaque `.jar` contient l'application et son serveur Tomcat. Il faut deux terminaux, un par microservice. Autre solution sans construire le `.jar` : `mvn spring-boot:run` dans chaque dossier.

**4. Un microservice est-il un service web ? Un service web est-il toujours un microservice ?**
Oui, un microservice est un service web : `calcul-service` et `historique-service` exposent des ressources REST accessibles par HTTP. Mais un service web n'est pas toujours un microservice. Le `RestWebApp.war` du TP2 était un service web, mais monolithique : calcul et historique dans une seule application, déployée dans un Tomcat séparé. Un microservice est petit, autonome, avec son propre serveur et son propre port, et il communique avec les autres par HTTP.

**5. Problème de l'adresse de `calcul-service` écrite dans `application.properties`**
L'adresse `http://localhost:8081` désigne une seule instance. Si on lance trois exemplaires de `calcul-service` (ports différents), `historique-service` n'appelle toujours que la première : la charge n'est pas répartie. Si cette instance tombe, il n'y a pas de basculement vers les autres. Il faudrait modifier la configuration à la main pour chaque changement. Une solution (TP4) : un annuaire de services et un répartiteur de charge.

**6. Stateless et `historique-service` qui conserve la liste des opérations**
Le principe stateless interdit de mémoriser l'**état du client** (session, contexte de conversation) entre deux requêtes. Chaque requête doit contenir toute l'information nécessaire pour être traitée. La liste des opérations est une **ressource** gérée par le serveur, partagée par tous les clients. Elle ne dépend d'aucun client en particulier, donc elle ne contredit pas le principe.

**7. POST répété et PUT répété**

- Deux fois `POST /operations` : deux opérations différentes sont créées (identifiants 1 et 2). POST n'est **pas idempotent**.
- Deux fois `PUT /operations/1` : l'opération 1 est remplacée par les mêmes valeurs, et l'état final est identique après un ou deux envois. PUT est **idempotent**.

La réponse n'est pas différente de celle du TP2 : l'idempotence est une propriété des verbes HTTP, pas de l'outil. Seule la façon de stocker les données a changé (attributs `static` au TP2, `@Service` singleton au TP3).
