## Installing AWS CLI

You will need to install the AWS CLI client using:

```
brew install awscli
```

Then you can configure your access by running:

```
aws configure
```

The credentials for this can be created using
https://console.aws.amazon.com/iam/home?region=us-east-1#/security_credentials.

## Example Build Process

This example adds GMP to the layer.

1. Install docker, (and update very important - I've got v. 2.1.3.0)
2. Clone https://github.com/salesmessage/vapor-php-build/
3. Run `composer install`
4. Find you extensions zip file (eg. https://gmplib.org/download/gmp/gmp-6.1.2.tar.xz)
5. Add version number to versions.ini file (gmp = "6.1.2")
6. Add instruction to install the extension (`php.Dockerfile`):
    ```
    # Build GMP

    ARG gmp
    ENV VERSION_GMP=${gmp}
    ENV GMP_BUILD_DIR=${BUILD_DIR}/gmp

    RUN set -xe; \
        mkdir -p ${GMP_BUILD_DIR}; \
        curl -Ls https://gmplib.org/download/gmp/gmp-${VERSION_GMP}.tar.xz \
        | tar xJC ${GMP_BUILD_DIR} --strip-components=1

    WORKDIR  ${GMP_BUILD_DIR}/

    RUN set -xe; \
        CFLAGS="" \
        CPPFLAGS="-I${INSTALL_DIR}/include  -I/usr/include" \
        LDFLAGS="-L${INSTALL_DIR}/lib64 -L${INSTALL_DIR}/lib" \
        ./configure \
            --prefix=${INSTALL_DIR} \
            --enable-shared \
            --disable-static

    RUN set -xe; \
        make install
    ```
7. Add `--with-gmp=shared,${INSTALL_DIR} \` to `php.Dockerfile` under `# Build PHP`
8. Add your extension to the /runtime/php.ini file.
9. Navigate into php folder and run `make distribution` (This takes a long time to compile)
10. Navigate into the root folder.
12. A file is added to your /exports folder.
13. Run php publish.php
14. Copy arn name, (In your region) and add it to your vapor.yml file as following (under layers not
    runtime!):
    ```
    environments:
        production:
            storage: xxxxxx
            memory: 1024
            cli-memory: 512
            database: xxxxx
            domain: xxxxx
            layers:
                - 'arn:aws:lambda:xxxxxxxx:xxxxxxx:layer:vapor-php-73:x'
            build:
                - 'composer install --classmap-authoritative'
                - 'php artisan event:cache'
                - 'php artisan config:cache'
                - 'php artisan view:cache'
                - 'php artisan route:cache'
            deploy:
                - 'php artisan migrate --force'
    ```