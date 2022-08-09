

# Runway Example php App

This is an example app demonstrating how to deploy a php app
to [runway](https://runway.planetary-quantum.com/).

* clone this repo, and navigate into that directory
* `runway app create`
* `runway app deploy`
* `runway open`

You can then deploy changes by `git commit`ing them, and running `runway app
deploy` again.

This is the [Symfony Demo Application](https://github.com/symfony/demo),
created with `composer create-project symfony/symfony-demo my_project`.

### PHP Extensions

By default, that demo application needs sqlite support, which has been
enabled by putting a `custom.ini` into the directory `.php.ini.d`, with the
following contents:
```ini
extension=pdo.so
extension=pdo_sqlite.so
```

### Webserver and PHP-FPM setup

We also configure some defaults for the buildpack in `project.toml`:
```toml
[ build ]
  [[ build.env ]]
    name="BP_PHP_SERVER"
    value="nginx" # we want nginx, with php-fpm
  [[ build.env ]]
    name="BP_PHP_ENABLE_HTTPS_REDIRECT"
    value="false" # no http-to-https redirects, the runway platform handles that
  [[ build.env ]]
    name="BP_PHP_WEB_DIR"
    value="public" # standard web directory for a symfony app
  [[ build.env ]]
    name="BP_COMPOSER_INSTALL_OPTIONS"
    value="" # reset composer options, standard is --no-dev
```

([more options are available](https://paketo.io/docs/howto/php/) but this is
 all we need for symfony)

We also tell nginx to fallback to the `index.php`, by putting a
`symfony-server.conf` into `.nginx.conf.d`:
```
location / {
    # try to serve file directly, fallback to index.php
    try_files $uri /index.php$is_args$args;
}
```

### Teaching symfony about symlinks

Buildpacks work in "layers", and because of that, `vendor/` is just a symlink
into a specific directory. Some symfony scripts don't like that. We fix that by
specifying the full path to `src` for the autoloader: 

```patch
--- a/content/php/composer.json
+++ b/content/php/composer.json
     },
     "autoload": {
         "psr-4": {
-            "App\\": "src/"
+            "App\\": "/workspace/src/"
         }
     },
     "autoload-dev": {
         "psr-4": {
-            "App\\Tests\\": "tests/"
+            "App\\Tests\\": "/workspace/tests/"
         }
     },
     "scripts": {
```

and explicitly setting the app's `root-dir` for symfony:

```patch
--- a/content/php/composer.json
+++ b/content/php/composer.json
     "extra": {
         "symfony": {
             "allow-contrib": true,
+            "root-dir": "/workspace",
             "require": "6.1.*"
         }
     }
 }
```

plus, we remove the post-install scripts, because these aren't run
in the context of the app and wouldn't work:

```patch
--- a/composer.json
+++ b/composer.json
@@ -87,7 +87,6 @@
             "assets:install %PUBLIC_DIR%": "symfony-cmd"
         },
         "post-install-cmd": [
-            "@auto-scripts"
         ],
         "post-update-cmd": [
             "@auto-scripts"

```

### Runtime Config

We also need to set `APP_ENV` to `prod` during runtime:
* `runway app config set APP_ENV=prod`

