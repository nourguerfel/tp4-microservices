# TP4 Spring Cloud : Microservices « Calculatrice »
*Guerfel Nour 
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

Captures : `screenshots/D4-test1.png` à `screenshots/D4-test7.png`

---

## Partie E : La passerelle gateway-service

**Question E.1.** Pourquoi l'en-tête `Location` indique-t-il le port 8888 et non 8082 ?

La passerelle ajoute des en-têtes `X-Forwarded-Host` et `X-Forwarded-Port` à la requête transmise à historique-service. Ils indiquent l'adresse réellement utilisée par le client. Grâce à `server.forward-headers-strategy=framework`, Spring lit ces en-têtes, et `ServletUriComponentsBuilder` construit l'URL avec `localhost:8888`.

### E.2 : Routes statiques en YAML

Tests via le Swagger UI de la passerelle (http://localhost:8888/swagger-ui.html).
Captures : `screenshots/E2-test1.png` à `screenshots/E2-test6.png`

(Tableau des 6 requêtes à compléter.)

### E.3 : Routes `lb://`

(À compléter.)

### E.4 : Routes en Java

(À compléter.)

### E.5 : Routes dynamiques

(À compléter.)

**Question E.2.** (à compléter)

**Question E.3.** (à compléter)

---

## Questions de synthèse

(À compléter : questions 1 à 11.)

---

## Captures d'écran

- Tableau de bord Eureka (3 services, CALCUL-SERVICE avec 2 instances) : `screenshots/eureka.png`
- Routes de la passerelle (configuration dynamique) : `screenshots/gateway-routes.png`
- Swagger UI de la passerelle, configuration statique : `screenshots/swagger-static.png`
- Swagger UI de la passerelle, configuration dynamique : `screenshots/swagger-dynamic.png`
- Les cinq consoles : `screenshots/consoles.png`
