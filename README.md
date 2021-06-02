# CRL - Certificate Revocation List

This repository contains the list of revoked certificates issued by openziti.org. This list is used
in conjunction with a proper x509 implementation to determine if the certificate provided is to be
considered valid.

## Generating Necessary Cryptographic Material

The following openssl commands generated the keys used by openziti.org to sign code. RSA was chosen for
the keys as EC is sometimes unsupported.  Commands sometimes reference a configuration file which is 
important. Running these commands should start by first cloning this repository and running the
commands from the root of the clone.

### Root Certificate Authority for openziti

The root certificate authority was generated and is valid for 7320 days (approx 20 years)

#### Private Key:
    openssl genrsa -out openziti.rootCA.rsa.key 4096

#### Certificate:
    openssl req \
        -new -x509 \
        -config ./openziti.openssl.conf \
        -key openziti.rootCA.rsa.key \
        -sha512 \
        -days 7320 \
        -subj "/CN=openziti.org Root Signing CA/O=NetFoundry Inc/OU=adv-dev" \
        -out openziti.rootCA.rsa.pem

### Code Signing Certificate for openziti

#### Private Key:
    openssl genrsa -out openziti.signing.rsa.key 4096

#### Certificate Signed by openziti.rootCA.rsa.pem

This is a two-part process of generating the CSR and then signing the CSR

    openssl req \
        -new -key openziti.signing.rsa.key \
        -subj "/CN=openziti.org 2021 Code Signing Certificate/O=NetFoundry Inc/OU=adv-dev" \
        -out openziti.signing.rsa.csr
    openssl x509  \
        -req  \
        -CA openziti.intermediary.rsa.pem \
        -CAkey openziti.intermediary.rsa.key  \
        -CAcreateserial \
        -days 366  \
        -sha512  \
        -in openziti.signing.rsa.csr  \
        -out openziti.signing.rsa.pem

### Creating a PKCS #12 File

The code signing tool of choice desires a PKCS#12 file to be supplied. A strong password was supplied

    openssl pkcs12 \
    -export \
    -inkey openziti.signing.rsa.key \
    -in openziti.signing.rsa.pem \
    -out openziti.signing.rsa.pfx 