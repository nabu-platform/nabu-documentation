@tags password protect

# Password Protect

Often you may want to shield your application from the public with a password, especially when it comes to publically available development or testing environments. 

By enabling the password protect, users will be greeted by a screen prompting them for a password before they can continue to the actual application. Once filled in correctly, the password is remembered on that device posing no further restrictions on the user.

[^.resources/password-protect-in-action.mp4]

## Adding it to your application

The ``nabu.web.core`` module comes with a component that you can plug into your application: ``nabu.web.core.passwordProtect``.

First we add the component to our application:

[^.resources/password-protect-add.mp4]

Once you have added it to your application, the configuration options for the password protect become available at the bottom of the web application. If you don't see them immediately, close and re-open the web application.

[^.resources/password-protect-configure.mp4]

If you don't fill in a password, the password protect is disabled. By enabling the environment specific toggle for the configuration you can choose how the password protect is configured per environment.

# Technical Details

## Security

You can choose a password, if you leave it open the password protect is disabled. If you set a specific [hashing algorithm](https://en.wikipedia.org/wiki/Cryptographic_hash_function), that will be used, otherwise it will default to [SHA256](https://en.wikipedia.org/wiki/SHA-2).

To prevent [man in the middle attacks](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) on an unsecured cookie over HTTP, you can choose to only allow access to the password protected content over HTTPS by enabling the ``Secure Only`` toggle.

A [salt|https://en.wikipedia.org/wiki/Salt_(cryptography)] is added in the backend to defend against a pre-computed hash attack.

## Overhead

Most hashing algorithms in the dropdown are fast and pose a negligible overhead cost. If you enable BCRYPT however, you can add as much as 300ms to every call being made.
