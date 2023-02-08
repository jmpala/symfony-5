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

- Creation of User entity class, the same implements "Symfony\Component\Security\Core\User\UserInterface"

```shell
php bin/console make:user
```

It added "app_user_provider" into "security.yaml" file. It is an object that knows how to load User objects from the database (or elsewhere). The provider is called by the firewall, at user authentication time or after each request, except when set "stateless: true" in the firewall configuration.

```yaml
security:
  # ...
  providers: # This key is always required, weather you use it or not
    app_user_provider:
      entity:
        class: App\Entity\User
        property: email
  # ...
```

- Creation of a Controller and its Login page

#### Login flow

- User enters email and password and submits the form
- Symfony receives the request and passes it to the firewall (Authenticators)
- Symfony Authenticators approves or rejects the request
- The requested controller is called

Firewalls are defined in the security.yaml file.

```yaml
security:
  # ...
  firewalls:
    dev:
      pattern: ^/(_(profiler|wdt)|css|images|js)/
      security: false
    main: # generic firewall
      lazy: true
      provider: app_user_provider
  #...
```

The firewalls are executed one by one and the first firewall that matches the request is used. The match is done by the pattern option (regex). Take in count that a pattern mishandling can lead to firewalls not being executed.

#### Authenticators

Symfony use the "Authenticator" system to authenticate users. It is a class that implements "Symfony\Component\Security\Http\Authenticator\AuthenticatorInterface".

We can create our own Authenticator with command:

```shell
php bin/console make:auth
```

In order to use the Authenticator, we need to add it to the "security.yaml" file.

```yaml
security:
  # ...
  firewalls:
    # ...
    main:
      # ...
      custom_authenticator: App\Security\AuthenticatorClass
```

The Authenticator class has the following methods:
- supports (check if the request is supported)
- authenticate
- onAuthenticationSuccess (executed if the authentication is successful)
- onAuthenticationFailure (executed if the authentication is not successful)

Authenticator use Passports and Badges to authenticate users (on authenticate method). The passport takes a UserBadge instance as first parameter and a CredentialsInterface as a second parameter. Because we defined an entity provider in the security.yaml file, the UserBadge will be automatically loaded from the database and then the credentials will be checked.

We can define our own custom Passport as follows:

```php
// SomeAuthenticator extends AbstractAuthenticator
// in the authenticate method
return new Passport(
            new UserBadge($email, function ($userIdentifier) { // does the user exist?
                $user = $this->userRepository->findOneBy(['email' => $userIdentifier]);

                if (!$user) {
                    throw new UserNotFoundException();
                }

                return $user;
            }),
            new CustomCredentials(
                function ($credentials, UserInterface $user) { // does he have valid credentials?
                    return password_verify($credentials, $user->getPassword());
                },
                $password
        ));
``` 

We can also define our own custom authentication failure behaviour:

```php
// SomeAuthenticator extends AbstractAuthenticator
// in the onAuthenticationFailure method
$request->getSession()->set(Security::AUTHENTICATION_ERROR, $exception);
return new RedirectResponse($this->router->generate('app_login'));
```

On the log-in we can throw the "UserNotFoundException". The same has a method called "getMessageKey" that gives us the message to show to the user, hiding sensible information. Symfony converts the exception into a "BadCredentialsException" upon configuration.

```yaml
security:
  # ...
  hide_user_not_found: false
```

The logout functionality is already implemented by Symfony. We can set the "logout" property to true in the security.yaml file. This will set a listener to the logout (by default "/logout") route and will clear the session.

```yaml
security:
    # ...
    firewalls:
        # ...
        main:
            # ...
            logout: true
```

### Authorization

#### Roles

