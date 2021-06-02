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
        -subj "/C=US/ST=NC/CN=openziti.org Root Signing CA/O=NetFoundry Inc/OU=adv-dev" \
        -out openziti.rootCA.rsa.pem

### Code Signing Certificate for openziti

#### Private Key:
    openssl genrsa -out openziti.signing.rsa.key 4096

#### Certificate Signed by openziti.rootCA.rsa.pem

This is a two-part process of generating the CSR and then signing the CSR

    openssl req \
        -new -key openziti.signing.rsa.key \
        -config ./openziti.openssl.conf \
        -subj "/C=US/ST=NC/CN=openziti.org 2021 Code Signing Certificate/O=NetFoundry Inc/OU=adv-dev" \
        -out openziti.signing.rsa.csr
    
    openssl ca -batch -config \
        ./openziti.openssl.conf \
        -keyfile openziti.rootCA.rsa.key \
        -cert openziti.rootCA.rsa.pem \
        -in openziti.signing.rsa.csr \
        -extfile openziti.openssl.conf \
        -out openziti.signing.rsa.pem


### Creating a PKCS #12 File

The code signing tool of choice desires a PKCS#12 file to be supplied. A strong password was supplied

    openssl pkcs12 \
        -export \
        -inkey openziti.signing.rsa.key \
        -in openziti.signing.rsa.pem \
        -out openziti.signing.rsa.pfx 

## Revoking a Certificate

A file exists in the repository named `certdb.txt`. This is a plaintext file that represents the revoked
certificates.  This file serves as the source of the actual CRL list: `openziti.crl`. It contains the 
serial numbers of revoked certificates as well as other information such as the date of revocation,
date and additional extensions which contain more details about the revoked certificates and the revocation reasons. The CRL also contains some global information attributes such as the version, signature algorithm, issuer name, issue date of the CRL and next update date.



Creating the CRL requires
access to the Root CA private key and thus cannot be performed by 'anyone'.