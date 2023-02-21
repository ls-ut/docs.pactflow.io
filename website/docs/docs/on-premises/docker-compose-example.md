---
title: Docker Compose example
---

In this guide, we'll provide an example `docker-compose` setup so that you can see how all of the bits hang together. We will demonstrate:

* Authenticating to Quay.io
* Running the PactFlow enterprise container
* Persistent storage with a postgres database
* Integrating to a test SAML provider

**Pre-requsites**
* [Docker](https://docs.docker.com/engine/install/)
* [Docker Compose](https://docs.docker.com/compose/install/)
* A working *nix environment and access to a terminal
* Valid credentials to authenticate to our [Docker image registry](docker-image-registry)
* PactFlow license file

## 1. Authenticating to Quay.io

After obtaining valid robot credentials, you need to authenticate to Quay.io so that the docker engine is able to fetch our images.

```
docker login -u="<username>" -p="<password>" quay.io
docker pull quay.io/pactflow/enterprise
```

After this, you should have the latest PactFlow enterprise image on your machine. You can verify by executing

```sh
docker images quay.io/pactflow/enterprise
```

Which should produce an output such as:

```sh
> docker images quay.io/pactflow/enterprise
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
quay.io/pactflow/enterprise   1.9.0               32db429fda01        7 weeks ago         454MB
quay.io/pactflow/enterprise   latest              32db429fda01        7 weeks ago         454MB
quay.io/pactflow/enterprise   1.8.0               7f9b3c3aa50e        3 months ago        462MB
```

## 2. PactFlow license file

The PactFlow on-premises version requires a license file to run. You should have received this from us during the
on-boarding process. If not, please contact us at support@pactflow.io.

Save the license file into a temporary directory (it needs to be the same directory as used in step 3).

## 3. Startup PactFlow and supporting services

Save the below file as `docker-compose.yml` into a temporary directory and then run `docker-compose up`:

```yaml
version: "3"

services:
  simplesaml:
    image: kristophjunge/test-saml-idp
    logging:
      driver: none # comment out the logging config to see the SAML server logs
    ports:
      - "8080:8080"
      - "8443:8443"
    environment:
     - SIMPLESAMLPHP_SP_ENTITY_ID=https://pactflow.io
     - SIMPLESAMLPHP_SP_ASSERTION_CONSUMER_SERVICE=http://localhost/auth/saml/callback

  pactflow:
    image: quay.io/pactflow/enterprise
    depends_on:
      - postgres
    environment:
      - PACTFLOW_HTTP_PORT=9292
      - PACTFLOW_BASE_URL=http://localhost
      - PACTFLOW_DATABASE_URL=postgres://postgres:password@postgres/postgres
      # insecure settings only for the purposes of this demo! Not to be used in production.
      - PACTFLOW_DATABASE_SSLMODE=disable
      - PACTFLOW_REQUIRE_HTTPS=false
      - PACTFLOW_LOG_FORMAT=short # normally this would be set to json, use short for demo only
      - PACTFLOW_ADMIN_API_KEY=admin
      - PACTFLOW_MASTER_SECRETS_ENCRYPTION_KEY=thisissomerandombytes
      - PACTFLOW_SAML_AUTH_ENABLED=true
      - PACTFLOW_SAML_IDP_NAME=Simple SAML
      - PACTFLOW_SAML_IDP_SSO_TARGET_URL=http://localhost:8080/simplesaml/saml2/idp/SSOService.php
      - PACTFLOW_SAML_IDP_CERT_FINGERPRINT=11:9B:9E:02:79:59:CD:B7:C6:62:CF:D0:75:D9:E2:EF:38:4E:44:5F
      - PACTFLOW_SAML_IDP_ID_ATTRIBUTE=uid
      - PACTFLOW_SAML_EMAIL_ATTRIBUTE=email
      - PACTFLOW_COOKIE_SECRET=thisisasecret
      - PACT_BROKER_ADMIN_API_KEY=admin
      - PACTFLOW_WEBHOOK_HOST_WHITELIST=/.*/
    ports:
      - "80:9292"
    healthcheck:
      test: ["CMD", "wget", "-nv", "-t1", "--spider", "http://localhost:9292/diagnostic/status/heartbeat"]
      interval: 30s
      timeout: 10s
      retries: 3
    entrypoint: dockerize
    command: -wait tcp://postgres:5432 docker-entrypoint
    volumes:
      - ./pactflow-onprem.lic:/home/pactflow-onprem.lic

  postgres:
    image: postgres
    healthcheck:
      test: psql postgres --command "select 1" -U postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-volume:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: postgres

volumes:
  postgres-volume:
```

You can verify all services by running `docker ps`:

```
CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS                             PORTS                                                    NAMES
8318130fa98a        quay.io/pactflow/enterprise   "docker-entrypoint"      18 seconds ago      Up 17 seconds (health: starting)   9292/tcp, 0.0.0.0:80->9292/tcp                           tmp_pactflow_1
7ba5d1679d09        kristophjunge/test-saml-idp   "docker-php-entrypoi…"   7 minutes ago       Up 17 seconds                      0.0.0.0:8080->8080/tcp, 80/tcp, 0.0.0.0:8443->8443/tcp   tmp_simplesaml_1
c0e3059fa37c        postgres                      "docker-entrypoint.s…"   7 minutes ago       Up 17 seconds (health: starting)   0.0.0.0:5432->5432/tcp                                   tmp_postgres_1
```

## 3. Login to PactFlow

Head to http://localhost in your browser, and choose to login with "SIMPLE SAML", with the username `user1` and password `user1pass`.

That's it 🎉.

You'll now want to do this using a proper container orchestration tool, such as Kubernetes, AWS ECS/Fargate, AKS, GKE, Mesos etc.