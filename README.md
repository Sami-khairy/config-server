Guide de Test : Rafraîchissement de la Configuration à Chaud
Ce guide explique comment tester le mécanisme de rafraîchissement dynamique de la configuration. Nous allons simuler un changement de secret pour un client d'administration Keycloak et vérifier que l'application prend en compte ce changement sans redémarrer.

Prérequis
Votre Config Server est en cours d'exécution.

Votre application multi-tenant (t-enant-app) est en cours d'exécution.

Vous avez un endpoint de test qui utilise le KeycloakAdminClientManager (par exemple, GET /api/admin/users/{username}).

Vous avez un jeton d'accès (token) valide pour un utilisateur du tenant_a.

Étape 1 : Vérifier que l'Application Fonctionne (État Initial)
Envoyez une requête à votre endpoint de test. Elle doit réussir.

# Remplacez VOTRE_TOKEN par un jeton valide pour le realm 'tenant_a'
TOKEN="VOTRE_TOKEN"

# Cette requête doit retourner une réponse 200 OK ou 404 Not Found (si l'utilisateur n'existe pas),
# mais PAS une erreur 500.
curl -i -X GET http://localhost:8080/api/admin/users/un_utilisateur_existant \
-H "X-TenantID: tenant_a" \
-H "Authorization: Bearer $TOKEN"

Résultat attendu : La requête réussit, prouvant que le client-secret actuel est correct.

Étape 2 : Modifier le Secret sur GitHub
Allez sur votre dépôt GitHub config-server.

Modifiez le fichier t-enant.properties.

Changez la valeur de la propriété tenants.admin-clients.tenant_a.client-secret. Mettez une valeur intentionnellement fausse, par exemple :

tenants.admin-clients.tenant_a.client-secret=ceci-est-un-mauvais-secret

Commitez et poussez (push) cette modification.

vous pouvez consulter http://localhost:8888/t-enant/default pour voir les nouveaux valeurs

Étape 3 : Déclencher le Rafraîchissement
Notifiez votre application qu'elle doit recharger sa configuration en appelant l'endpoint /actuator/refresh.

curl -X POST http://localhost:8080/actuator/refresh

Résultat attendu : Vous devriez recevoir une réponse JSON listant les clés qui ont été modifiées. La clé tenants.admin-clients.tenant_a.client-secret doit apparaître dans cette liste.

Étape 4 : Vérifier que l'Application a Échoué (État Final)
Relancez exactement la même requête que dans l'étape 1.

# Utilisez le même token et la même commande qu'à l'étape 1
curl -i -X GET http://localhost:8080/api/admin/users/un_utilisateur_existant \
-H "X-TenantID: tenant_a" \
-H "Authorization: Bearer $TOKEN"

Résultat attendu : La requête doit maintenant échouer, très probablement avec une erreur 500 Internal Server Error. En regardant les logs de votre application, vous verrez une erreur d'authentification (401 Unauthorized) lorsque le KeycloakAdminClientManager essaiera de se connecter à Keycloak avec le mauvais secret.

Ce test prouve que :

Votre application a bien rechargé la nouvelle propriété.

Le KeycloakAdminClientManager a été recréé (grâce à @RefreshScope).

Il utilise maintenant la nouvelle configuration (qui est fausse), ce qui provoque l'échec attendu.

Vous avez ainsi validé l'ensemble du processus de rafraîchissement à chaud.
