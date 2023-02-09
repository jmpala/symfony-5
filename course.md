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

In case of Login we can use the "AbstractLoginFormAuthenticator" that has already generic login logic implemented. The same can be configured inside "security.yaml" file.

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

We can also use the "PasswordCredentials" class to automatically check the given passwords of a user with the one saved on the database.

The Passport gives us the possibility to register more Badges on its third parameter. Here we can send an array of Badges that we want to register.

```php
return new Passport(
            new UserBadge(/* some params */),
            new PasswordCredentials(/* some params */),
            [
                new CsrfTokenBadge('string', $csrfToken),
                (new RememberMeBadge())->enable(), // we enable the REMEMBERME cookie
                // more Badges
            ]
);
```

And on the form we just add the following code:

```html
<!-- Twig will generate a Token for us -->
<input type="hidden" name="_csrf_token" value="{{ csrf_token('authenticate') }}">
```

We can register the "RememberMeBadge" into the "Passport", to set a "REMEMBERME" cookie to the user's browser. This will keep the user logged-in even if the "PHPSESSID" is lost/erased. One option to set this cookie is to let the user request it through a form. The name of this input should be "_remember_me" or else we need to change the required name by the "RememberMeBadge" through the "security.yaml" file.

```html
<label>
    <input type="checkbox" name="_remember_me" class="form-check-input">Remember me
</label>
```

Or we can tell Symfony through the "security.yaml" file to always set the "REMEMBERME" cookie. The "signature_properties" let us configure which User values to use to generate the cookie signature. To for example, If we change the password, the cookie will be invalidated.

```yaml
security:
    #...
    firewalls:
        # ...
        main:
            # ...

            remember_me:
                # always_remember_me: true
                signature_properties: [password]
```

We can also define our own custom authentication failure behaviour:

```php
// SomeAuthenticator extends AbstractAuthenticator
// in the onAuthenticationFailure method
$request->getSession()->set(Security::AUTHENTICATION_ERROR, $exception);
return new RedirectResponse($this->router->generate('app_login'));
```

To show error messages on the FE, we have to our disposal the service "AuthenticationUtils". With the same, we can retrieve the last authentication error and the last username entered by the user.

```php
// AuthenticationUtils is injected into the controller
$authenticationUtils->getLastAuthenticationError()
$authenticationUtils->getLastUsername()
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

There are several ways to handle Authorization within Symfony. We can configure it through:

- "security.yaml" file
- controller annotations
- Twig templates

#### Assign roles hierarchy

In the "security.yaml" file we can define the roles hierarchy. The roles are defined as an array of strings. The first role is the parent role and the following are the children roles.

```yaml
security:
    # ...
    role_hierarchy:
        ROLE_ADMIN: [ROLE_USER]
        ROLE_SUPER_ADMIN: [ROLE_ADMIN, ROLE_ALLOWED_TO_SWITCH]
        ROLE_PARENT: [ROLE_CHILD_1, ROLE_CHILD_2]
```

#### Examples of authorization in controller

Checking User roles by method:

```php
/**
     * @Route("/questions/new")
     */
    public function new()
    {
        $this->denyAccessUnlessGranted('ROLE_USER'); // by method
        return new Response('Sounds like a GREAT feature for V2!');
    }
```

Checking User roles by annotation:

```php
/**
     * @Route("/questions/new")
     * @IsGranted("ROLE_USER") // by annotation
     */
    public function new()
    {
        return new Response('Sounds like a GREAT feature for V2!');
    }
```

The application always throw an "AccessDeniedException" when the user is not authorized to access the requested page.

#### Examples of authorization in Twig

Checking User roles by method:

```html
{% if is_granted('ROLE_USER') %}
    <a href="{{ path('app_action') }}">Link</a>
{% endif %}
```

#### Different types of roles

- "ROLE_USER" - A regular user
- "PUBLIC_ACCESS" - A user that is not logged-in
- "IS_AUTHENTICATED_FULLY" - A user that has been authenticated fully (logged in)
- "IS_AUTHENTICATED_REMEMBERED" - A user that has been remembered via the "REMEMBERME" cookie

#### Redirect the User, when access requested page and not logged-in

We can set the redirection to a page through:

- "security.yaml" file, property "entry_point", indicating an Authenticator
- or directly at one Authenticator, implementing "AuthenticationEntryPointInterface"

Example of "security.yaml" file:

```yaml
security:
    # ...
    firewalls:
        # ...
        main:
            # ... We select only one of the two Authenticators options
            entry_point: App\Security\LoginFormAuthenticator
            custom_authenticator:
                - App\Security\LoginFormAuthenticator
                - App\Security\DummyAuthenticator
```

Example of Authenticator:

```php
// SomeAuthenticator implements AuthenticationEntryPointInterface
public function start(Request $request, AuthenticationException $authException = null): Response
{
    return new RedirectResponse('app_login');
}
```

#### User Impersonation

Symfony gives us the possibility to impersonate other user accounts in order to experience our application through their perspective. We can enable this functionality by modifying the "security.yaml" file.

```yaml
security:
    # ...
    firewalls:
        # ...
        main:
            # ...
            switch_user: true
```

In order to impersonate a user, we need to be logged-in as an admin (with role "ROLE_ALLOWED_TO_SWITCH"). We can do this as follows:

- by accessing the URL "/_switch_user/{username}"
- or through a query string "?switch_user={username}"

This will set a "SWITCH_USER" cookie in the browser. The same will be used to impersonate the user. In order to exit the impersonation mode, we can replace the username with "_exit". 
