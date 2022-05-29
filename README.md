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

Store files generated below offline and in a secure place.

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

    docker-compose up -d

Or manually:

    docker run \
      --rm \
      --network pki \
      --name ca_intermediate \
      --volume ca_intermediate:/var/lib/cfssl \
      --env AUTH_KEY="000102030405060708090a0b0c0d0e0f" \
      1nfiniteloop/pki pki_serve_ca ca_intermediate

Edit csr for server, example can be found in folder [csr/](csr/):

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

Start pki cron service to renew certificate when expired. Argument to `pki_cron`
is the name(s) of the certificates to be checked daily and renewed:

    docker run \
      --rm \
      --network pki \
      --name pki-cron \
      --user root \
      --volume pki_certs:/var/lib/cfssl \
      --env CFSSL_CA_SIGNING_PROFILE=server_cert \
      --env CFSSL_CA_REMOTE=ca_intermediate \
      --env AUTH_KEY="000102030405060708090a0b0c0d0e0f" \
      1nfiniteloop/pki pki_cron server.lan

### Certificate usage

Servers:

To allow other services to read the certificates it is recommended that
the user of the running the service add itself to group "pki" with gid 500.
It is also necessary to allow group "pki" to read the private key:

    docker run \
      --rm \
      --volume pki_certs:/var/lib/cfssl \
      1nfiniteloop/pki
      chmod g+r server.lan-key.pem

Clients:

All hosts that interacts with the tls certifcates needs the root ca certificate
installed, else tls connections will fail with "certificate verify failed (self
signed certificate in certificate chain)". To automate the root certificate
provisioning, serve root certificate over http with:

    docker run \
      --rm \
      -it \
      --name pki-ca-cert \
      -u root \
      --volume ca_intermediate:/var/lib/cfssl:ro \
      1nfiniteloop/pki mini_httpd -D -C /etc/mini_httpd/mini_httpd.conf

Then download and install root certificate on each hosts that interacts with the
tls certificates:

    curl -o /usr/local/share/ca-certificates/ca_cert.crt \
      http://<pki-ca-cert>/ca_cert.pem
    update-ca-certificates

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

    openssl verify -CAfile ca_intermediate-ca-chain.pem server.lan.pem

## Reference

* <https://github.com/cloudflare/cfssl>
* <https://www.rfc-editor.org/rfc/rfc5280>
