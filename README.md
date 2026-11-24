# TP4 Spring Cloud : Microservices « Calculatrice »
# Guerfel Nour

**Matière :** Architecture MicroServices, MPSRCC1
**Technologies :** Spring Boot 4.0.9, Eureka, OpenFeign, Spring Cloud Gateway, Swagger UI, Lombok

## Structure du projet

```
calculatrice-spring-cloud/
|-- pom.xml                 (pom parent)
|-- eureka-service/         (annuaire, port 8761)
|-- calcul-service/         (calculs, ports 8081 et 8083)
|-- historique-service/     (opérations, port 8082)
`-- gateway-service/        (passerelle, port 8888)
```

Ordre de lancement : eureka-service, calcul-service, historique-service, gateway-service.

---

## Partie A : Questions de conception

**Question 1.** Quel composant fait la correspondance entre le nom et les adresses ? Quand est-elle mise à jour ?

C'est l'annuaire Eureka (Eureka Server). Il garde la liste des instances de chaque service. La liste est mise à jour quand une instance s'inscrit au démarrage, quand elle se désinscrit à l'arrêt, ou quand elle est retirée parce qu'elle n'envoie plus son battement de cœur. Les clients en gardent une copie locale, rafraîchie régulièrement (toutes les 5 secondes dans ce TP).

**Question 2.** Un avantage et un inconvénient du point d'entrée unique.

- Avantage : le client ne connaît qu'une seule adresse (le port 8888). On peut ajouter, déplacer ou remplacer des microservices sans changer le client.
- Inconvénient : si la passerelle tombe en panne, plus rien n'est accessible. Elle peut aussi devenir un goulot d'étranglement, car toutes les requêtes passent par elle.

**Question 3.** Six requêtes `GET /calcul/somme?a=1&b=1` avec deux instances.

Chaque instance reçoit 3 requêtes, car la répartition se fait à tour de rôle (round robin). Le client ne peut pas s'en apercevoir : la réponse est toujours la même (2).

---

## Partie B : L'annuaire eureka-service

**Question B.1.** Pourquoi Eureka peut-il tourner avec calcul-service et historique-service ?

Chaque programme a son propre port : 8761 pour Eureka, 8081 pour calcul-service, 8082 pour historique-service. Il n'y a donc aucun conflit de port.

---

## Partie C : Enregistrer les microservices dans l'annuaire

**Question C.1.** Les deux instances ont le même nom mais des identifiants différents. À quoi sert chacun ?

Le nom (`CALCUL-SERVICE`) est le même pour toutes les instances : un service qui veut appeler calcul-service n'a besoin que de ce nom. L'identifiant (`calcul-service:8081`, `calcul-service:8083`) désigne une instance précise. Il permet à Eureka et au répartiteur de charge de distinguer les instances et d'en choisir une.

**Question C.2.** Que se passerait-il sans l'option `--server.port=8083` ?

La seconde instance utiliserait le port 8081 (lu dans `application.properties`), déjà pris par la première. Le démarrage échouerait avec l'erreur « Port 8081 was already in use ».

**Expérience C.6 (arrêt d'une instance).**

Quand l'instance est arrêtée proprement (bouton Stop), elle prévient Eureka et disparaît tout de suite de l'annuaire. Si le processus est tué brutalement, Eureka ne reçoit plus de battement de cœur et retire l'instance après environ 10 secondes. Quand on relance l'instance, elle réapparaît dès son démarrage.

---

## Partie D : OpenFeign

**Question D.1.** Six fois la requête 1 : quelle instance répond ? Quelle règle ? Quel nom ?

Les requêtes vont alternativement à 8081, 8083, 8081, 8083, 8081, 8083. Chaque instance en reçoit trois. La règle est : « prendre l'instance suivante de la liste, à tour de rôle ». Cette stratégie s'appelle le **round robin**.

**Question D.2.** Nom du service mal écrit (`CALCUL-SERVIC`).

On obtient le statut **503 Service Unavailable**. Eureka ne connaît aucun service avec ce nom, donc Feign échoue. La réponse est produite par le bloc `catch (FeignException e)` (le bloc général), avec le message « calcul-service injoignable ». Le bon nom a ensuite été remis.

### Résultats des tests D.4 (via Swagger UI, port 8082)

| N° | Requête | Corps | Statut | Instance qui a tracé |
|---|---|---|---|---|
| 1 | POST /operations | `{"operateur":"somme","a":8,"b":2}` | 201 | (à compléter) |
| 2 | POST /operations | `{"operateur":"produit","a":6,"b":7}` | 201 | (à compléter) |
| 3 | POST /operations | `{"operateur":"division","a":7,"b":2}` | 201 | (à compléter) |
| 4 | POST /operations | `{"operateur":"division","a":7,"b":0}` | 400 | (à compléter) |
| 5 | POST /operations | `{"operateur":"racine","a":9,"b":0}` | 400 | aucune (erreur détectée avant l'appel) |
| 6 | GET /operations | aucun | 200 | aucune |
| 7 | GET /operations/2 | aucun | 200 | aucune |

---

## Partie E : La passerelle gateway-service

**Question E.1.** Pourquoi l'en-tête `Location` indique-t-il le port 8888 et non 8082 ?

La passerelle ajoute des en-têtes `X-Forwarded-Host` et `X-Forwarded-Port` à la requête transmise à historique-service. Ils indiquent l'adresse réellement utilisée par le client. Grâce à `server.forward-headers-strategy=framework`, Spring lit ces en-têtes, et `ServletUriComponentsBuilder` construit l'URL avec `localhost:8888`.

### E.2 : Routes statiques en YAML (adresses fixes)

Quatre routes : `/calcul/**` vers `http://localhost:8081`, `/operations/**` vers `http://localhost:8082`, et deux routes `/docs/...` pour Swagger UI. Tests faits depuis le Swagger UI de la passerelle (http://localhost:8888/swagger-ui.html).

| N° | Requête (préfixe http://localhost:8888) | Statut | Réponse | Qui a répondu |
|---|---|---|---|---|
| 1 | GET /calcul/somme?a=2&b=3 | 200 | 5 | calcul-service 8081 |
| 2 | GET /calcul/division?a=7&b=0 | 400 | Erreur : division par 0 impossible | calcul-service 8081 |
| 3 | GET /operations | 200 | tableau des opérations de D.4 | historique-service |
| 4 | POST /operations `{"operateur":"soustraction","a":10,"b":4}` | 201 | resultat 6.0, Location avec le port 8888 | historique-service, puis calcul-service |
| 5 | GET /calcul/somme/detail?a=2&b=3 | 200 | `{"operation":"somme","a":2,"b":3,"resultat":5.0}` | calcul-service 8081 |
| 6 | GET /bonjour (navigateur) | 404 | erreur JSON de la passerelle | la passerelle (aucune route) |

Observation : avec une adresse fixe, la trace sort toujours sur l'instance 8081. L'instance 8083 est ignorée.

### E.3 : Routes `lb://` (via l'annuaire)

Les quatre adresses fixes sont remplacées par `lb://CALCUL-SERVICE` et `lb://HISTORIQUE-SERVICE`. La passerelle demande à Eureka les instances du service et les sollicite à tour de rôle. Après six clics sur `GET /calcul/somme?a=2&b=3`, les traces alternent entre 8081 et 8083. Le Swagger UI fonctionne toujours.

### E.4 : Routes en Java

Les mêmes routes sont déclarées dans un bean `RouteLocator` dans `GatewayServiceApplication`. Les deux formes (YAML et Java) sont équivalentes, et si les deux sont présentes, leurs routes s'ajoutent. Les six requêtes de E.2 donnent les mêmes résultats.

### E.5 : Routes dynamiques (créées à partir de l'annuaire)

La passerelle crée une route par service enregistré dans Eureka, grâce à `discovery.locator.enabled: true`. Le nom du service devient le premier segment de l'URL, et la passerelle l'enlève avant de transmettre la requête.

| N° | Requête (préfixe http://localhost:8888) | Requête transmise | Statut | Réponse |
|---|---|---|---|---|
| 1 | GET /calcul-service/calcul/somme?a=2&b=3 | GET /calcul/somme?a=2&b=3 | 200 | 5 |
| 2 | GET /calcul-service/calcul/produit/6/7 | GET /calcul/produit/6/7 | 200 | 42 |
| 3 | GET /historique-service/operations | GET /operations | 200 | tableau des opérations |
| 4 | POST /historique-service/operations `{"operateur":"somme","a":1,"b":1}` | POST /operations | 201 | resultat 2.0 |
| 5 | GET /calcul/somme?a=2&b=3 (navigateur) | aucune | 404 | plus de route `/calcul/**` |
| 6 | GET /gateway-service/actuator/gateway/routes (navigateur) | vers la passerelle elle-même | 200 | liste JSON des routes |

La page `http://localhost:8888/actuator/gateway/routes` liste une route par service de l'annuaire. Quand on arrête historique-service et qu'on attend une dizaine de secondes, sa route disparaît. Quand on le relance, elle réapparaît : les routes suivent l'annuaire.

**Question E.2.** Comparez les URI des étapes E.3 et E.5. Quel est l'avantage de chaque forme ? Laquelle oblige à modifier la passerelle quand on ajoute un microservice ?

- E.3 (routes statiques `lb://`) : les URI vues du client sont propres (`/calcul/somme`) et ne montrent pas le nom interne du service. On choisit librement les chemins et les filtres de chaque route.
- E.5 (routes dynamiques) : on n'écrit aucune route. Un nouveau microservice enregistré dans Eureka est accessible tout de suite, sans toucher à la passerelle.
- La forme **statique (E.3)** oblige à modifier la passerelle (ajouter une route) à chaque nouveau microservice.

**Question E.3.** La route 6 montre que la passerelle a une route vers elle-même. Pourquoi ? Est-ce gênant ?

La passerelle est aussi un client Eureka : elle s'inscrit dans l'annuaire sous le nom `GATEWAY-SERVICE`. Comme le mécanisme dynamique crée une route pour **tous** les services de l'annuaire, il en crée aussi une pour la passerelle elle-même. Ce n'est pas gênant pour le TP. En production, ça peut exposer des URL internes (comme les endpoints Actuator), donc on peut filtrer les services pour lesquels des routes sont créées.

---

## Questions de synthèse

**1. Tableau A.3 : étape où la différence a été observée.**

| Aspect | Étape du TP |
|---|---|
| Nombre de programmes | B (annuaire), C.5 (2e instance), E.1 (passerelle) |
| Dépendances Maven | B.2, C.1, C.7, E.1 |
| Adresse d'un microservice | C.2 et D.2 (le nom remplace l'adresse) |
| Appel entre microservices | D.2 (interface `CalculClient`) |
| Erreurs d'appel | D.3 et question D.2 (`FeignException`, 503) |
| Point d'entrée du client | E.2 (tout passe par le port 8888) |
| Plusieurs instances d'un service | C.5, D.4 et E.3 (traces qui alternent 8081 / 8083) |
| Serveur embarqué | B.5 (Tomcat) et E.2 (Netty : « Netty started on port 8888 ») |
| Arrêt d'un microservice | C.6 |
| Organisation du code | B.0 et C.0 (un projet, un module par service) |
| Outil de test | C.4, D.4, E.2 (Swagger UI) |
| Classes de données | C.1 bis et C.7 (Lombok) |

**2. Trajet de `POST :8888/historique-service/operations` avec `{"operateur":"somme","a":1,"b":2}`.**

1. Le client (Swagger UI) envoie la requête à la **passerelle** (8888).
2. La passerelle trouve la route dynamique `/historique-service/**`. Elle consulte la copie locale de l'annuaire Eureka pour connaître les instances de `HISTORIQUE-SERVICE`, et son LoadBalancer en choisit une. Elle enlève le préfixe et ajoute les en-têtes `X-Forwarded-*`.
3. **Requête HTTP 1** : `POST /operations` vers historique-service (8082).
4. `OperationController` appelle `calculClient.calculer("somme", 1, 2)`. OpenFeign consulte la copie locale de l'annuaire pour `CALCUL-SERVICE`, et LoadBalancer choisit une instance (8081 ou 8083).
5. **Requête HTTP 2** : `GET /calcul/somme?a=1&b=2` vers calcul-service. La réponse est `3`.
6. historique-service range l'opération (résultat 3.0) et répond **201** avec l'en-tête `Location`.
7. La réponse remonte : historique-service, passerelle, client.

Au total il y a 3 requêtes HTTP (client vers passerelle, passerelle vers historique, historique vers calcul). Il y a 2 consultations de l'annuaire : une par la passerelle pour `HISTORIQUE-SERVICE`, une par historique-service pour `CALCUL-SERVICE`.

**3. Les trois programmes avec Spring Cloud LoadBalancer.**

Le répartiteur est inclus dans le client Eureka, donc il est présent dans les trois services :

- `calcul-service` : il ne résout aucun nom, car il n'appelle personne.
- `historique-service` : il résout `CALCUL-SERVICE` (via OpenFeign).
- `gateway-service` : il résout `CALCUL-SERVICE` et `HISTORIQUE-SERVICE` (via les routes `lb://` ou dynamiques).

**4. Client Eureka, OpenFeign et Spring Cloud Gateway : où sont-ils nécessaires ?**

| Dépendance | calcul-service | historique-service | gateway-service |
|---|---|---|---|
| Client Eureka | Oui : pour s'inscrire dans l'annuaire | Oui : pour s'inscrire et trouver calcul-service | Oui : pour trouver les services et s'inscrire |
| OpenFeign | Non : il n'appelle aucun autre service | Oui : il appelle calcul-service par son nom | Non : la passerelle transmet avec ses routes |
| Spring Cloud Gateway | Non | Non | Oui : c'est la passerelle elle-même |

**5. Réponse complète à la question 5 du TP3.**

Avec l'adresse écrite en dur, historique-service appelle toujours la même instance de calcul-service, celle dont l'adresse est dans sa configuration. Même si on lance trois exemplaires, les deux autres ne reçoivent jamais de requête, donc la charge n'est pas répartie. De plus, si l'adresse change, il faut modifier la configuration et relancer le service. Dans ce TP, le problème est résolu par :

- **Eureka Server** : il garde la liste de toutes les instances en marche.
- **Eureka Client** : chaque instance s'inscrit toute seule et envoie son battement de cœur.
- **OpenFeign** : l'appel se fait par le nom `CALCUL-SERVICE`, plus par une adresse.
- **Spring Cloud LoadBalancer** : il choisit une instance à chaque appel, à tour de rôle.
- **Spring Cloud Gateway** (`lb://`) : elle répartit aussi la charge pour les requêtes des clients.

**6. Deux instances d'historique-service (8082 et 8084) avec des opérations en mémoire.**

Les trois POST sont répartis entre les deux instances (par exemple 2 sur l'une et 1 sur l'autre). Chaque instance a sa propre mémoire. `GET /historique-service/operations` renvoie donc seulement les opérations de l'instance qui répond, et le résultat change d'un appel à l'autre (2 opérations, puis 1, puis 2...). Les id peuvent aussi se répéter.

Le principe mis en défaut est celui de **service sans état** (stateless) : les instances d'un même service doivent être interchangeables. Pour corriger, on place les données dans une **base de données partagée** (MySQL, PostgreSQL...) ou dans un cache partagé comme Redis, que toutes les instances utilisent.

**7. Construire et lancer les cinq programmes sans IntelliJ.**

À la racine de `calculatrice-spring-cloud` (il faut Maven installé), une seule commande construit les quatre modules :

```
mvn clean package -DskipTests
```

Puis, dans cinq terminaux, dans cet ordre :

```
java -jar eureka-service/target/eureka-service-0.0.1-SNAPSHOT.jar
java -jar calcul-service/target/calcul-service-0.0.1-SNAPSHOT.jar
java -jar calcul-service/target/calcul-service-0.0.1-SNAPSHOT.jar --server.port=8083
java -jar historique-service/target/historique-service-0.0.1-SNAPSHOT.jar
java -jar gateway-service/target/gateway-service-0.0.1-SNAPSHOT.jar
```

**8. Le rôle de `@GetMapping("/calcul/{operateur}")` dans `CalculController` et dans `CalculClient`.**

Dans `CalculController` (côté serveur), l'annotation décrit la requête **reçue** : Spring envoie vers cette méthode les requêtes `GET /calcul/...` qui arrivent. Dans `CalculClient` (côté client), elle décrit la requête à **envoyer** : OpenFeign s'en sert pour construire la requête HTTP quand on appelle la méthode. L'annotation est la même, mais lue dans l'autre sens.

**9. Le regroupement en modules remet-il en cause l'indépendance ?**

Non. Chaque microservice reste indépendant :

- **JVM** : chacun tourne dans son propre processus Java.
- **Port** : chacun a son propre port (8761, 8081, 8082, 8888).
- **Fichier .jar** : chacun a son propre `pom.xml` et produit son propre .jar.
- **Cycle de vie** : on peut lancer, arrêter ou redéployer un service sans toucher aux autres (voir C.6 : arrêter une instance ne coupe pas les autres).

Le pom racine ne sert qu'à ranger le code et à construire les modules d'un coup : ce n'est même pas leur parent Maven.

**10. D'où vient l'adresse de la zone Servers de Swagger UI ? Que se passe-t-il sans `server.forward-headers-strategy=framework` ?**

L'adresse vient des en-têtes `X-Forwarded-Host`, `X-Forwarded-Port` et `X-Forwarded-Prefix` ajoutés par la passerelle. Le microservice les lit et les met dans sa description OpenAPI. En E.2 il n'y a pas de préfixe, donc on obtient `http://localhost:8888`. En E.5 la passerelle enlève le préfixe `/calcul-service` et l'indique dans `X-Forwarded-Prefix`, donc on obtient `http://localhost:8888/calcul-service`.

Sans la propriété, le microservice ignore ces en-têtes et annonce sa propre adresse (par exemple `http://localhost:8081`). Le Execute de Swagger UI enverrait alors la requête directement au microservice, **sans passer par la passerelle**. En E.5, le préfixe serait perdu et les requêtes tomberaient en erreur 404.

**11. Où se trouvent les constructeurs, getters et setters générés par Lombok ? Pourquoi Lombok peut-il être exclu du .jar ?**

Ils se trouvent dans le fichier compilé (par exemple `Operation.class` dans `target/classes`), pas dans le code source. Lombok est un processeur d'annotations : il ajoute ces méthodes au bytecode pendant la compilation. Une fois le `.class` produit, il contient déjà toutes les méthodes. Lombok n'est donc plus nécessaire à l'exécution, et on l'exclut du .jar pour l'alléger.
