# Consol Webinar: Application Security with OAuth2 and Keycloak

## What we will do
We start by scratch with a blank keycloak docker image. Throughout this session, we will
- Create and understand realms
- Create clients
- Create users
- Query and verify tokens through REST calls
- Define client-specific mappings
- Understand and create roles
- Understand the use-case for groups, create groups

## Out of scope
- A discussion of default clients and roles provided by keycloak
- A discussion of different OAuth-flows
- The use of refresh tokens
- How to use JWT Tokens in the backend system for Authorization

## Tools needed
- A functional docker installation
- Internet connection (for the initial pull of the docker images)
- A tool to execute REST calls. We will use the command-line tool [`curl`][curl], but one can 
achieve the same with, for example, [Insomnia][insomnia] or [Postman][postman].

## Starting the docker deployment

To start playing around, we need a running keycloak instance and a database as backing persistence 
provider. For this, we have created a docker-compose file in the [`./docker`][dockerFolder] folder.
The deployment can be started by changing into the directory and executing `docker-compose`:

    cd ./docker
    docker-compose up -d

The initial download of the docker image can take some time, so feel free to grab a coffee.

After the images have been pulled, keycloak itself takes some time to start up. We can check the 
status of the container by executing

    docker ps
    CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                              NAMES
    98fd842caeea        jboss/keycloak:10.0.2   "/opt/jboss/tools/do…"   14 minutes ago      Up 14 minutes       8443/tcp, 0.0.0.0:8090->8080/tcp   docker_keycloak-service_1
    12f40ec3282e        postgres:12.2           "docker-entrypoint.s…"   14 minutes ago      Up 14 minutes       5432/tcp                           docker_postgres-service_1

and then showing the log of the keycloak container:

    docker logs -f 9
    ...
    07:36:09,979 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0025: Keycloak 10.0.2 (WildFly Core 11.1.1.Final) started in 21244ms - Started 690 of 995 services (708 services are lazy, passive or on-demand)

When we see the above log message, that means keycloak is up and running, ready to be configured.

## Logging in to the web UI
The keycloak container is configured to listen on local port 8090, so accessing 
[http://localhost:8090][localKeycloak]. We are greeted by the keycloak login page:

![KeycloakHome][keycloakHome]

A click on `Administration Console` brings us to the actual login:

![KeycloakLogin][keycloakLogin]

The keycloak is configured with the following credentials:

![KeycloakAdminConsole][keycloakAdminConsole]

- Username: `keycloak`
- Password: `keycloak`

After the successful login, we see the adminstrative web ui:

## Realms
Realms are a means to coarsely subdivide your userbase.

As we can see on the upper left, we are currently on the `Master` realm. This is a special realm to 
manage the keycloak configuration. To manage our users, we want to create a new realm. We can 
achieve this by hovering over `Master`, revealing a menu to `Add Realm`.

![keycloakNewRealm][keycloakNewRealm]

For this demo, we will call our realm `app-realm`. After clicking on `Create`, the realm will be 
created and its base configuration opened:

![keycloakRealmConfig][keycloakRealmConfig]

Here, we can configure the newly created realm. For example, under the tab `Login`, we can configure
whether users are able to create new accounts or reset their password.

## Clients
Clients are entities that use the keycloak server to get authentication information, e.g. self-owned
microservices or other websites. To limit access to the authentication information, the access is 
restricted. Whenever a services wants to interact with keycloak, it must provide a client id and, 
depending on the operation, a secret to identify itself. Keycloak comes with some clients pre-defined 
that are necessary for keycloak to fuction. We can see the clients by clicking on `Clients` under 
`Configs` in the left menu:

![keycloakClients][keycloakClients]


We will ignore the existing clients for now and instead create two new clients `serviceOne` and 
`serviceTwo`:

![keycloakClients][keycloakClients]

The two roles will represent two different services that need different user information. We will 
see later on how we can customize those information, based on the client.

Since we are going to use the `password grant` for this demonstration, we need to configure both 
newly create clients to allow them this grant. 

In the detail view of the client, weswitch the `Access Type` to `confidential`, disable the 
Standard Flow and click `Save` on the bottom of the page.

![keycloakConfigureClient][keycloakConfigureClient]

After saving, a new tab is `Credentials` is visible. We switch to that tab and write down the 
secret, we will use it later on.

Repeat the process for both clients.

## Users, Groups and Roles

### Users
The core concept of authentication are users and roles. A user represents an entity interacting with 
the services - a human being. We can create a user by clicking on the menu `Users` under `Manage` on
the left of the screen.

![keycloakUsers][keycloakUsers]

The list show will always be empty. If we had created some users already, we would have to click the
button `View all users`.

To create a new user, we simply click `Add user` on the upper right: 

![keycloakAddUser][keycloakAddUser]

We just fill in the username as `alice` and then click `Save`. This will create the user and show an
overview of the user. Notice, however that we have not yet defined a password for the user. To set a 
password, we click on the tab `Credentials` and set a new password (`password`) for alice. We want 
the password to not be temporary, so we have to click the matching toggle button:

![setUserPassword][setUserPassword]

#### Fetch the first token
Now that we have our first user, we can try to fetch our first token by executing a `curl` command 
from the console. Please remember to replace `<serviceOneSecretHere>` with the actual secret of 
client `serviceOne`.

    curl \
      -k \
      --data "grant_type=password&client_id=serviceOne&client_secret=<serviceOneSecretHere>&username=alice&password=password" \
      http://localhost:8090/auth/realms/app-realm/protocol/openid-connect/token

The response should look something like this:

    {
        "access_token":"eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICI5M1NoMFl6anFBU0xYWUYwdmxuQ29IamhjZTl1VVlnV3VrSk1leWxNQlpFIn0.eyJleHAiOjE1OTI1NjIxMTgsImlhdCI6MTU5MjU2MTgxOCwianRpIjoiMGJlNWVmNTctNWY4My00NDJkLWFkNjEtNTc3OTRkMTUxOTkxIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDkwL2F1dGgvcmVhbG1zL2FwcC1yZWFsbSIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiJjZDJhYzY1YS01NzY5LTQ5NjItOTc4Zi1hNTFhYjU5YWRlZjciLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJzZXJ2aWNlT25lIiwic2Vzc2lvbl9zdGF0ZSI6IjQ3OGMxMWIwLTUzZDUtNDA5MC1hODM5LWI2Yzk0NDUzN2M4OCIsImFjciI6IjEiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwicHJlZmVycmVkX3VzZXJuYW1lIjoiYWxpY2UifQ.T5EUOr0vEYyMC2nMbs5cc3XD3Lh8Ek6pYyga7PI88lL13Z-vK_a4znxKYxnMabZZGzjNI6hM1VBiUFIrfoqCQkEpfeN4Ye2QQpfy4XeH6--_830cYcSY7SBhdxD0uE3w84gHiPApieMlwsY2MQ6sSdr65OnH8odqEtBNpRhfEOP-FkpXoNrHTE4B8cKIsEYt9csf27ykAmqYDvHcyzJK6uJq9IQ5PekBYlAguAt6vEuEDrVSHddDUbjmL7csMpqM_D6yeklh9gZbnNKeLOOdxvNXdda-p62KM69wId4OjBAf36-a7dOXaszEUSWaK9l-Xahhp3_LCbhjAlZ-DFGdhw",
        "expires_in":300,
        "refresh_expires_in":1800,
        "refresh_token":"eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJhMWM4ZTRhNy0zNDNlLTRiOTItYWI5Yy1mMzg4YTlmMDFhM2IifQ.eyJleHAiOjE1OTI1NjM2MTgsImlhdCI6MTU5MjU2MTgxOCwianRpIjoiMmU5MTQzYTgtYTAxNC00NjE0LThjYTktMDNiMzg1OTcwYmFmIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDkwL2F1dGgvcmVhbG1zL2FwcC1yZWFsbSIsImF1ZCI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA5MC9hdXRoL3JlYWxtcy9hcHAtcmVhbG0iLCJzdWIiOiJjZDJhYzY1YS01NzY5LTQ5NjItOTc4Zi1hNTFhYjU5YWRlZjciLCJ0eXAiOiJSZWZyZXNoIiwiYXpwIjoic2VydmljZU9uZSIsInNlc3Npb25fc3RhdGUiOiI0NzhjMTFiMC01M2Q1LTQwOTAtYTgzOS1iNmM5NDQ1MzdjODgiLCJzY29wZSI6ImVtYWlsIHByb2ZpbGUifQ.6AyyRT4b0YNRasIKzLt5yIzA212-2c6e_fyliNJ7nCE",
        "token_type":"bearer",
        "not-before-policy":0,
        "session_state":"478c11b0-53d5-4090-a839-b6c944537c88",
        "scope":"email profile"
    }

(The response shown here is formatted for clarity)

We see that the response includes two tokens, one called `access_token` and one called 
`bearer_token`, as well as some meta-inforation.

The `access_token` is the actual authentication provider. It is a Base64-encoded string. We can
decode it either manually or let the website [jwt.io][jwt.io] do the decoding for us.

The payload-part of jwt.io is the interesting bit. It shows the following information:

    {
      "exp": 1592562118,
      "iat": 1592561818,
      "jti": "0be5ef57-5f83-442d-ad61-57794d151991",
      "iss": "http://localhost:8090/auth/realms/app-realm",
      "aud": "account",
      "sub": "cd2ac65a-5769-4962-978f-a51ab59adef7",
      "typ": "Bearer",
      "azp": "serviceOne",
      "session_state": "478c11b0-53d5-4090-a839-b6c944537c88",
      "acr": "1",
      "realm_access": {
        "roles": [
          "offline_access",
          "uma_authorization"
        ]
      },
      "resource_access": {
        "account": {
          "roles": [
            "manage-account",
            "manage-account-links",
            "view-profile"
          ]
        }
      },
      "scope": "email profile",
      "email_verified": false,
      "preferred_username": "alice"
    }

Each key-value pair of this JSON-object is called a *claim*.

We can see when the token was issued (`iat`), when it expires (`exp`), the issuer of the token 
(`iss`), the client that made the request (`azp`) and the entity authenticated by this token 
(`preferred_username`), along with furhter meta-information. If we now, for example, add an email 
address to `alice` and then request a new token, the token will then include the email address:

    {
      ...
      "email": "alice@wonder.land"
    }

Furthermore, we can see that there are two `roles`-objects: the one is `realm_access.roles`, the 
other is `resource_access.account.roles`. `realm_access.roles` lists the roles of the entity 
authenticated (alice), while `resource_access.account.roles` shows the roles of the client that made
the query.

#### Modifying the token

Suppose that we do not want the tokens to include the client roles. How do we do that? The 
attributes stored in keycloak must be mapped to token claims. This done through *mappers*. Mappers, 
in return, are bound to clients or client scopes. Client scopes can be bound to clients.

Inspecting the mappers for client `serviceOne` reveals that this client does not have any mapper 
associated:

![keycloakClientMapper][keycloakClientMapper]

But `serviceOne` has some client scopes associated:

![keycloakClientScope][keycloakClientScope]

In particular, we see the `roles` scope associated.

Inspecting the mappers associated with the client scope `roles` shows a mapper `client roles`:

![keycloakClientScopeMapper][keycloakClientScopeMapper]

Deleting this mapper from the client scope and issuing a new token will give us the intended result:

    {
      "exp": 1592564972,
      "iat": 1592564672,
      "jti": "04c12559-da70-4578-8611-aae6ab747825",
      "iss": "http://localhost:8090/auth/realms/app-realm",
      "aud": "account",
      "sub": "cd2ac65a-5769-4962-978f-a51ab59adef7",
      "typ": "Bearer",
      "azp": "serviceOne",
      "session_state": "d7477175-21b8-4b15-837c-91af7b050a73",
      "acr": "1",
      "realm_access": {
        "roles": [
          "offline_access",
          "uma_authorization"
        ]
      },
      "scope": "email profile",
      "email_verified": false,
      "preferred_username": "alice",
      "email": "alice@wonder.land"
    }

This removes the claim `resource_access.account.roles` for *all* clients. But what if we want to 
only modify information for *one specific* client? In this case, we can add our own mapper. 
Suppose, for example, that for `serviceOne`, we want to add the value `service.one.example.com` to 
the audience (`aud`) claim to identify that only this service should use the token. For this, we 
navigate to the mappers of client `serviceOne` and click the `Create` button on the upper right. We
define the mapper to add the value we want to the audience claim:

![keycloakAddMapperToClient][keycloakAddMapperToClient]

Fetching and inspecting a new token reveals that, in deed, the value has been added to the audience
claim:

    {
      ...
      "aud": [
        "service.one.example.com",
        "account"
      ],
      ...
    }

If we, however, issue a token for client `serviceTwo` with the following command (remember to 
replace `<serviceTwoSecretHere>` with the actual secret):

    curl \
      -k \
      --data "grant_type=password&client_id=serviceTwo&client_secret=2f2d5ef2-875a-48d2-b1d4-1a905ae32312&username=alice&password=password" \
      http://localhost:8090/auth/realms/app-realm/protocol/openid-connect/token

and inspect this token, we see that `"service.one.example.com"` is missing from the audience claim:

    {
      "exp": 1592565846,
      "iat": 1592565546,
      "jti": "84bcd0ff-87a3-45cd-b405-f4a6eb826369",
      "iss": "http://localhost:8090/auth/realms/app-realm",
      "aud": "account",
      "sub": "cd2ac65a-5769-4962-978f-a51ab59adef7",
      "typ": "Bearer",
      "azp": "serviceTwo",
      "session_state": "e811a9d1-e0af-403f-b3d1-054925cc8843",
      "acr": "1",
      "realm_access": {
        "roles": [
          "offline_access",
          "uma_authorization"
        ]
      },
      "scope": "email profile",
      "email_verified": false,
      "preferred_username": "alice",
      "email": "alice@wonder.land"
    }
    
Thus, we have a client-dependent token mapping.

### Roles

Roles define which actions a user is allowed to execute. These information is used by the backend 
system to allow/deny actions or access to web resources. 

alice already has some roles (`"offline_access"` and `"uma_authorization"`) associated. We can add
new roles by clicking on the corresponding menu on the left, and then clicking `Add Role`: 

![keycloakRoles][keycloakRoles]

Let's keep things simple and add two roles: `User` and `Admin`. Then, let us add those roles to the 
user `alice`:

![keycloakUserAddRole][keycloakUserAddRole] 

If we now request and inspect a new token for alice, we see that both roles are added to the token:

    {
      ...
      "realm_access": {
        "roles": [
          "User",
          "offline_access",
          "uma_authorization",
          "Admin"
        ]
      },
      ...
    }

### Groups
If we have some more roles or maybe dependencies between roles (e.g. "each `Admin` should also be a 
`User`), we can create groups, assign the roles to the group and then assign the group to a user. 
What is more: we can define default groups that are always added to a new user. This gives us an 
elegant solution to manage complex role structures.

We start by adding two Groups `Users` and `Admins`. Next, we add the roles `User` and `Admin` to the 
group `Admins`:

![keycloakGroupAddRoles][keycloakGroupAddRoles]

For the group `Users`, we only add the role `User`.

Next, we define the group `Users` as default group:

[keycloakDefaultGroups][keycloakDefaultGroups]

If we now create a new user `bob`, the new user will have the group `Users` associated:

![keycloakUserGroups][keycloakUserGroups]

and what's more, `bob` also inherited the roles from the group:

![keycloakUserRoles][keycloakUserRoles]

And, of course, if we now issue a token for `bob` (we have to set passwort for `bob` first, though),
the roles will be visible in the token:

    {
      "exp": 1592567504,
      "iat": 1592567204,
      "jti": "7e690009-3cf9-43fe-a39c-c6f3fcc52453",
      "iss": "http://localhost:8090/auth/realms/app-realm",
      "aud": [
        "service.one.example.com",
        "account"
      ],
      "sub": "f2b7eb22-b0d8-4c83-bf35-101cba3ed1e7",
      "typ": "Bearer",
      "azp": "serviceOne",
      "session_state": "dbab8222-d214-4e60-94d4-89d8e2f844f0",
      "acr": "1",
      "realm_access": {
        "roles": [
          "User",
          "offline_access",
          "uma_authorization"
        ]
      },
      "scope": "email profile",
      "email_verified": false,
      "preferred_username": "bob"
    }

[curl]: https://curl.haxx.se/
[insomnia]: https://insomnia.rest/
[postman]: https://www.postman.com/
[dockerFolder]: ./docker
[localKeycloak]: http://localhost:8090/auth
[keycloakHome]: ./images/KeycloakHome.png
[keycloakLogin]: ./images/KeycloakLogin.png
[keycloakAdminConsole]: ./images/KeycloakAdminConsole.png
[keycloakNewRealm]: ./images/KeycloakNewRealm.png
[keycloakRealmConfig]: ./images/KeycloakRealmConfig.png
[keycloakClients]: ./images/KeycloakClients.png 
[keycloakUsers]: ./images/KeycloakUsers.png
[keycloakAddUser]: ./images/KeycloakAddUser.png
[setUserPassword]: ./images/SetUserPassword.png
[keycloakClients]: ./images/KeycloakNewClient.png
[keycloakConfigureClient]: ./images/KeycloakConfigureClient.png
[jwt.io]: https://jwt.io/
[keycloakClientMapper]: ./images/KeycloakClientMapper.png
[keycloakClientScope]: ./images/KeycloakClientScope.png
[keycloakClientScopeMapper]: ./images/KeycloakClientScopeMapper.png
[keycloakAddMapperToClient]: ./images/KeycloakAddMapperToClient.png
[keycloakRoles]: ./images/KeycloakRoles.png
[keycloakUserAddRole]: ./images/KeycloakUserAddRole.png
[keycloakGroupAddRoles]: ./images/KeycloakGroupAddRoles.png
[keycloakDefaultGroups]: ./images/KeycloakDefaultGroups.png
[keycloakUserGroups]: ./images/KeycloakUserGroups.png
[keycloakUserRoles]: ./images/KeycloakUserRoles.png