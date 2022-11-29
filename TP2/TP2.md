# TP2 - Catherine Vanden Hende

## Etape 1

### Deployer l'api

Les pods ne peuvent pas se lancer correctement car ils leur manque une connexion à postgres :

```bash
***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to bind properties under 'spring.datasource.hikari.username' to java.lang.String:

    Property: spring.datasource.hikari.username
    Value: ${POSTGRES_USER}
    Origin: class path resource [application.yml]:9:17
    Reason: Could not resolve placeholder 'POSTGRES_USER' in value "${POSTGRES_USER}"

```

## Etape 3

### Lier l'API et la BDD

Le nom du service de la base de données est pg-service

## Etape 4

### Front

Le computer n'a pas été enregistré en base et comme on a changé de pod, il a été perdu.

## Etape 5

### Persistance

Le type d'accès pour le Storage Class EBS est ReadWriteOnce d'après le tableau. Cela convient bien à la persistance de norte bdd car elle ne peut etre lue ou modifiée que par 1 pod à la fois donc notre volume restera cohérent.

Le nom du PV est :
pvc-b6345c6d-7998-4e35-b678-1a6d7fe56b24
