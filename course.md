# Symfony Security

```shell
composer require security
```

## Security system

- Authentication Manager (new System)

Divided into 2 parts:
- Authentication
- Authorization

### Authentication

1. Creation of User entity class, the same implements "Symfony\Component\Security\Core\User\UserInterface"

```shell
php bin/console make:user
```

It added a "app_user_provider" into "security.yaml" file. It is an object that knows how to load User objects from the database (or elsewhere).

```yaml
security:
    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email
```

2. Creation of a Controller and its Login page

### Authorization

#### Roles

