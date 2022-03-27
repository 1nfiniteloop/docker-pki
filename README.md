# pki

This tool is used to build your own pki infrastructure. It uses cfssl
(see <https://github.com/cloudflare/cfssl>) and is distributed as a docker
container.

The instructions below covers how to setup a root -and intermediate
ca and how to issue client/server certificates. It also contain scripts to
renew certs when expired. This tools is intended to be used together with other
docker services that requires tls certificates.

## Usage

__Note:__ Make sure to replace the secret key in env `AUTH_KEY` in commands
below.

### Create root ca

Store files generated below for root ca offline and in a secure place.

Create csr for root ca. Example can be found in folder [csr/](csr/):

    vi csr/ca_root.csr.json

Create key, csr and certificate for root ca. The generated files will be
prefixed with `ca_root`:

    cat csr/ca_root.csr.json \
      |docker run -i \
      --rm \
      --user $(id -u):$(id -g) \
      --volume path/to/secure/storage:/var/lib/cfssl \
      1nfiniteloop/pki pki_init_ca_root ca_root

### Create intermediate ca

Create a network for pki traffic, used in steps below:

    docker network create pki

Start root ca container and serve api endpoints for signing intermediate csr.

    docker run \
      --rm \
      --env AUTH_KEY="000102030405060708090a0b0c0d0e0f" \
      --name ca_root \
      --network pki \
      --user $(id -u):$(id -g) \
      --volume path/to/secure/storage:/var/lib/cfssl \
      1nfiniteloop/pki pki_serve_ca ca_root

Edit csr for intermediate ca, example can be found in folder [csr/](csr/):

    vi csr/ca_intermediate.csr.json

Create intermediate csr and sign remotely with root ca. Env `CFSSL_CA_REMOTE`
is the fqdn of the remote ca server. Env `AUTH_KEY` must be equal to key
in root ca. Env `CFSSL_CA_SIGNING_PROFILE` selects profile, see
`overlay/etc/cfssl/config.ca.json`.  The generated files will be
prefixed with `ca_intermediate`:

    cat csr/ca_intermediate.csr.json \
      |docker run -i \
      --rm \
      --network pki \
      --volume ca_intermediate:/var/lib/cfssl \
      --env CFSSL_CA_REMOTE=ca_root \
      --env CFSSL_CA_SIGNING_PROFILE=ca \
      --env AUTH_KEY="000102030405060708090a0b0c0d0e0f" \
      1nfiniteloop/pki pki_init_cert ca_intermediate

### Issue new cert from intermediate ca

Start intermediate ca service:

    docker run \
      --rm \
      --network pki \
      --name ca_intermediate \
      --volume ca_intermediate:/var/lib/cfssl \
      --env AUTH_KEY="000102030405060708090a0b0c0d0e0f" \
      1nfiniteloop/pki pki_serve_ca ca_intermediate

Edit csr for intermediate ca, example can be found in folder [csr/](csr/):

    vi csr/server.lan.csr.json

Init pki server cert:

    cat csr/server.lan.csr.json \
      |docker run -i \
      --rm \
      --network pki \
      --volume cert:/var/lib/cfssl \
      --env AUTH_KEY="000102030405060708090a0b0c0d0e0f" \
      --env CFSSL_CA_SIGNING_PROFILE=server_cert \
      --env CFSSL_CA_REMOTE=ca_intermediate \
      1nfiniteloop/pki pki_init_cert server.lan

Start pki cron service to renew certificate when expire. Argument to `pki_cron`
is the name(s) of the certificates to be checked and renewed:

    docker run \
      --rm \
      --network pki \
      --name pki-cert \
      --user root \
      --volume pki_certs:/var/lib/cfssl \
      --env CFSSL_CA_SIGNING_PROFILE=server_cert \
      --env CFSSL_CA_REMOTE=ca_intermediate \
      --env AUTH_KEY="000102030405060708090a0b0c0d0e0f" \
      1nfiniteloop/pki pki_cron server.lan

If the volume is shared with another container that uses the certificate it
might require uid/gid of user "pki" to be adjusted:

    docker exec -u root -it pki-cert usermod --uid 300 pki
    docker exec -u root -it pki-cert groupmod --gid 300 pki
    docker exec -u root -it pki-cert chown -R pki:pki ${CFSSL_CERT_PATH}

## Inspect

The files created from commands above can be inspected with openssl:

Inspect key:

    openssl pkey -noout -text -in ca-key.pem

Inspect cert:

    openssl x509 -noout -text -in ca.pem

Inspect csr:

    openssl req -noout -text -in ca.csr

Verify intermediate certificate:

    openssl verify -CAfile ${ca_root}/ca_root.pem ca_intermediate.pem

Verify server/client certificates:

    openssl verify -CAfile ca_intermediate-bundle.pem server.lan.pem

## Reference

* <https://github.com/cloudflare/cfssl>
* <https://www.rfc-editor.org/rfc/rfc5280>
