Run Mendix in Cloud Foundry
=====

There are specific guides for deploying Mendix apps to the [Pivotal Web Services](https://world.mendix.com/display/howto50/Deploying+a+Mendix+App+to+Pivotal) and [HP Helion Development Platform](https://world.mendix.com/display/howto50/Deploying+a+Mendix+App+to+HP+Helion) flavors of Cloud Foundry on our [documentation page](https://world.mendix.com/display/howto50/Deploying+a+Mendix+App+to+Cloud+Foundry). This page will document the more low-level details and CLI instructions.


Deploying using the CLI
----


### Install cloud foundry command line

Install the Cloud Foundry command line executable. You can find this on the [releases page](https://github.com/cloudfoundry/cli#stable-release). Set up the connection to your preferred Cloud Foundry environment with `cf login` and `cf target`.


### Push your app

We push an mda (Mendix Deployment Archive) that was built by the Mendix Business Modeler to Cloud Foundry.

    cf push <YOUR_APP> -b https://github.com/mendix/cf-mendix-buildpack -p <YOUR_MDA>.mda

We can also push a project directory. This will move the build process (using mxbuild) to Cloud Foundry:

    cd <PROJECT DIR>; cf push -b https://github.com/mendix/cf-mendix-buildpack

Note that building the project in Cloud Foundry takes more time and requires enough memory in the compile step.


### Configuring admin password

The first push generates a new app, but deployment will fail because the buildpack requires an `ADMIN_PASSWORD` variable and a connected PostgreSQL or MySQL service. So go ahead and set this up after the first failed push.

Keep in mind that the admin password should comply with the policy you have set in the Modeler.

    cf set-env <YOUR_APP> ADMIN_PASSWORD "<YOURSECRETPASSWORD>"


### Connecting a Database

You also need to connect a PostgreSQL or MySQL instance which allows at least 5 connections to the database. Find out which services are available in your Cloud Foundry instance like this.

    cf marketplace

In our trial we found the service `elephantsql` which offered the free `turtle` plan. All you need to do is give it a name and bind it to your application.

    cf create-service elephantsql turtle <SERVICE_NAME>

    cf bind-service <YOUR_APP> <SERVICE_NAME>

Note that not all database service set a `DATABASE_URL` value. If this is not done automatically you need to set this variable manually using the details included in the service, as the buildpack will look for this variable for the database connection string.

Now we need to push the application once more.

    cf push <YOUR_APP> -b https://github.com/mendix/cf-mendix-buildpack -p <YOUR_MDA>.mda

You can now log in to your application with the specified password.


### Configuring Constants

The default values for constants will be used as defined in your project. However, you can override them with environment variables. You need to replace the dot with an underscore and prefix it with `MX_`. So a constant like `Module.Constant` with value `ABC123` could be set like this:

    cf set-env <YOUR_APP> MX_Module_Constant "ABC123"

After changing environment variables you need to restart your app. A full push is not necessary.

    cf restart <YOUR_APP>


### Configuring Scheduled Events

The scheduled events can be configured using environment variable `SCHEDULED_EVENTS`.

Possible values are `ALL`, `NONE` or a comma separated list of the scheduled events that you would like to enable. For example: `ModuleA.ScheduledEvent,ModuleB.OtherScheduledEvent`


### Configuring External Filestore

Mendix 5.15 and up can use external file stores with an S3 api. Use the following environment variables to enable this.

* `S3_ACCESS_KEY_ID`: credentials access key
* `S3_SECRET_ACCESS_KEY`: credentials secret
* `S3_BUCKET_NAME`: bucket name

The following environment variables are optional:
* `S3_PERFORM_DELETES`: set to `false` to never delete items from the filestore. This is useful when you use the filestore without a backup mechanism.
* `S3_KEY_SUFFIX`: if your bucket is multi-tenant you can append a string after each object name, you can restrict IAM users to objects with this suffix.
* `S3_ENDPOINT`: for S3 itself this is not needed, for S3 compatible object stores set the domain on which the object store is available.
* `S3_USE_V2_AUTH`: use Signature Version 2 Signing Process, this is useful for connecting to S3 compatible object stores like Riak-CS, or Ceph.
* `S3_ENCRYPTION_KEYS`: a [JSON string](example-s3-encryption-keys.json) containing a list of keys which can be used to encrypt and decrypt data at rest in S3.
* `S3_USE_SSE`: if set to `true` this will enable Server Side Encryption in S3, available from Mendix 6.0


### Configuring the Java heap size

The default java heap size is set to the total available memory divided by two. If your application's memory limit is 1024M, the heap size is set to 512M. You might want to tweak this to your needs by using another environment variable in which case it is used directly.

    cf set-env <YOUR_APP> HEAP_SIZE 512M


### Configuring the Java version

The default Java version is 7. If you want to run on Java 8, you can set the environment variable `JAVA_VERSION` to `8`.

    cf set-env <YOUR_APP> JAVA_VERSION 8


### Enabling the Mendix Debugger

You can enable the Mendix Debugger by setting a `DEBUGGER_PASSWORD` environment variable. This will enable and open up the debugger for the lifetime of this process and is to be used with caution. The debugger is reachable on https://DOMAIN/debugger/. You can follow the second half of this [How To](https://world.mendix.com/display/howto50/Debugging+Microflows+Remotely) to connect with the Mendix Business Modeler. To stop the debugger, unset the environment variable and restart the application.


### Enabling sticky sessions

Mendix apps in version 5.15 and up will automatically set `JSESSIONID` as the session cookie. In most Cloud Foundry configurations this will automatically enable session stickiness, which is required for running Mendix apps with more than one instance. For some distributions you might need to explicitly enable session stickiness in the HTTP router.

If you want to disable the session stickiness, you can set the environment variable `DISABLE_STICKY_SESSIONS` to `true`.


### Configuring Cluster Support (BETA feature for Mendix 6)

From Mendix 6 onwards it is possible to configure clustering support. With clustering support it is possible to run multiple instances of your application to achieve High Availability. Clustering support can be enabled by setting the environment variable `CLUSTER_ENABLED` to `true`.

To optimize performance it is possible to choose for REDIS as State Storage. To use that, add the `RedisCloud` service to your application and configure Mendix to use REDIS for State Storage by setting the environment variable `CLUSTER_STATE_IMPLEMENTATION` to `redis`. In case your selected REDIS service has limited connections available, configure the maximum amount of connections that REDIS can allocate per Mendix Business Server instance by setting the environment variable `CLUSTER_STATE_REDIS_MAX_CONNECTIONS` to the amount of connections available in your plan divided by the amount of instances you want to run.

NOTE: Clustering support is a BETA feature and as such not supported by Mendix for production usage. Settings and implementations may be changed in future releases!

NOTE: Enabling clustering support will automatically disable sticky sessions


### Offline buildpack settings

If you are running Cloud Foundry without a connection to the Internet, you should specify an on-premises web server that hosts Mendix Runtime files and other buildpack resources. You can set the endpoint with the following environment variable:

`BLOBSTORE: https://my-intranet-webserver.my-company.com/mendix/`

The preferred way to set up this on-premises web server is as a transparant proxy to `https://cdn.mendix.com/`. This prevents manual work by system administrators every time a new Mendix version is released.

Alternatively you can make the required mendix runtime files `mendix-VERSION.tar.gz` available under `BLOBSTORE/runtime/`. You should also make the Java version available on `/mx-buildpack/oracle-java8u45-jdk_8u45_amd64.deb` available under `BLOBSTORE/mx-buildpack/`. The original files can be downloaded from `https://cdn.mendix.com/`.


### Certificate Management

To import Certificate Authorities (CAs) into the Java truststore, use the `CERTIFICATE_AUTHORITIES` environment variable.

The contents of this variable should be a concatenated string containing a the additional CAs in PEM format that are trusted. Example: 

    -----BEGIN CERTIFICATE-----
    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
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    MIIEcTCCA1mgAwIBAgIJANUE5069bkdvMA0GCSqGSIb3DQEBBAUAMIGBMQswCQYD
    VQQGEwJOTDEVMBMGA1UECBMMWnVpZC1Ib2xsYW5kMRIwEAYDVQQHEwlSb3R0ZXJk
    YW0xEjAQBgNVBAoTCU1lbmRpeCBCVjESMBAGA1UEAxMJTWVuZGl4IENBMR8wHQYJ
    KoZIhvcNAQkBFhBiZWhlZXJAbWVuZGl4Lm5sMB4XDTA1MTEzMDA5MjY1NFoXDTE1
    MTEyODA5MjY1NFowgYExCzAJBgNVBAYTAk5MMRUwEwYDVQQIEwxadWlkLUhvbGxh
    bmQxEjAQBgNVBAcTCVJvdHRlcmRhbTESMBAGA1UEChMJTWVuZGl4IEJWMRIwEAYD
    VQQDEwlNZW5kaXggQ0ExHzAdBgkqhkiG9w0BCQEWEGJlaGVlckBtZW5kaXgubmww
    ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCizJkE35dyDUpz2GgZzVdZ
    Rlf/eA9Xe48hx3WFe2sLGO42ngb71qSQutDxfYStCjT17/25JH5URLfTX/9L4WFe
    INj6uX+Lt8W1ODtgVJ+HvgoJH76etpcXggOLsX8GFXhAdZWiwZ7S3rlVJiaVJWSc
    VrZZkzXwK9Y/la4HjGHGVyd52doYBXb3uMJt9Fl1daT7cz11WTTlUiQEHfkRfROZ
    KXN0o7JtBZqwrHsKaloYPfoW/9SlmrlAe4WJV1+WsdPpxzjfI730lpBgaY6XsLHT
    3I+l/BYHJZx+8jBUFhi+0Aj9TX2Xx3Ran7dmB5dezCyLzcgpM31WE3gid+ELzdFd
    AgMBAAGjgekwgeYwHQYDVR0OBBYEFMZSioCtznEB4WO1S/c7x+4VI2YWMIG2BgNV
    HSMEga4wgauAFMZSioCtznEB4WO1S/c7x+4VI2YWoYGHpIGEMIGBMQswCQYDVQQG
    EwJOTDEVMBMGA1UECBMMWnVpZC1Ib2xsYW5kMRIwEAYDVQQHEwlSb3R0ZXJkYW0x
    EjAQBgNVBAoTCU1lbmRpeCBCVjESMBAGA1UEAxMJTWVuZGl4IENBMR8wHQYJKoZI
    hvcNAQkBFhBiZWhlZXJAbWVuZGl4Lm5sggkA1QTnTr1uR28wDAYDVR0TBAUwAwEB
    /zANBgkqhkiG9w0BAQQFAAOCAQEAKdXKdTLlCPfn4vkJzTg+ukD2CSQysTx+24+P
    4BSUKZ+lOHBmhsKia1Zs+upvbHZ7605x2cdpppEiKC+aVQJJZ2X3BrZZq25oYdDg
    Z1LiFonMl7o4oOjVXhVix4T/WbxuGZTLxpdPHJA+SGw7yaA5Fh0uT70bjTeIfVS6
    cTYdfWO9rsrhiSYt4YeCarDCnjO93vxInvog3ydjJB69luTUXXoniaEnFEXPSqqN
    EVSQHw0FN1bNvaA6/zXvdb7E2oFKIYiPylsdeQ6DPgBxht/YMZRlI8p3F2SiEbbe
    yW7wMeYCUfgTNWaSaJd6uYUjj+IP/9+YOkp5pLW5eEAq6YscYA==
    -----END CERTIFICATE-----

Please note, these are two internal Mendix CAs which you should not actually add to your trust store.


Data Snapshots
====

If you want to enable initializing your database and files from an existing data snapshot included in the MDA, set `USE_DATA_SNAPSHOT` to `true`.


Contributing
====

Make sure your code complies with pep8 and that no pyflakes errors/warnings are present.

Rebase your git history in such a way that each commit makes one consistent change. Don't include separate "fixup" commits later on.
