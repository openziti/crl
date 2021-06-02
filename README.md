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
        -subj "/CN=Root Signing CA/O=openziti.org Inc/OU=adv-dev/C=US/ST=NC" \
        -out certs/openziti.rootCA.rsa.pem

### Code Signing Certificate for openziti

#### Private Key:
    openssl genrsa -out openziti.signing.rsa.key 4096

#### Certificate Signed by openziti.rootCA.rsa.pem

This is a two-part process of generating the CSR and then signing the CSR. Replace the year appropriately

    openssl req \
        -new -key openziti.signing.rsa.key.torevoke \
        -config ./openziti.openssl.conf \
        -subj "/CN=Code Signing Certificate 2021/O=openziti.org Inc/OU=adv-dev/C=US/ST=NC" \
        -out openziti.signing.rsa.csr.torevoke
    
    openssl ca \
        -batch \
        -config ./openziti.openssl.conf \
        -keyfile openziti.rootCA.rsa.key \
        -cert certs/openziti.rootCA.rsa.pem \
        -days 1098 \
        -in openziti.signing.rsa.csr.torevoke \
        -extfile openziti.signing.rsa.conf \
        -out certs/openziti.signing.2021.rsa.pem.torevoke;

### Creating a PKCS #12 File

The code signing tool of choice desires a PKCS#12 file to be supplied. The following command is run to produce a PKCS#12
file. During the process a password prompt is generated and a strong password was supplied.

    openssl pkcs12 \
        -export \
        -inkey openziti.signing.rsa.key \
        -in certs/openziti.signing.rsa.pem \
        -out openziti.signing.rsa.pfx 

### Verify The Signing Certificate

Once generated, issue the following command and verify the certificate contains the expected extensions:

    #command to execute:
    openssl x509 -text -in certs/openziti.signing.2021.rsa.pem

    #expected extensions below:
    X509v3 extensions:
        X509v3 Authority Key Identifier:
            keyid:4E:9A:FF:21:B7:03:42:BF:15:6B:ED:43:82:16:71:3C:FF:40:93:E4

        X509v3 Basic Constraints:
            CA:FALSE
        X509v3 Key Usage:
            Digital Signature
        X509v3 Extended Key Usage:
            Code Signing
        Netscape CA Revocation Url:
            https://openziti.github.io/crl/revoked.pem
        Netscape Revocation Url:
            https://openziti.github.io/crl/revoked.pem

## Signing Code Using Signtool

If you need to sign an executable in Windows you would use signtool. Here's an example command illustrating how to 
sign an executable with a signing certificate. %PFXPASS_OPENZITI% is an environment variable set in `cmd`:

    signtool sign /f openziti.signing.rsa.pfx /p %PFXPASS_OPENZITI% /fd sha512 /td sha512 executable_name.exe

## Revoking a Certificate

A file exists in the repository named `certdb.txt`. This is a plaintext file that represents the revoked
certificates.  This file serves as the source of the actual CRL list: `openziti.crl`. It contains the
serial numbers of revoked certificates as well as other information such as the date of revocation.

Creating the CRL requires access to the Root CA private key and thus can only be performed by authorized personnel.

To revoke a certificate perform the following steps:

1. clone this repository and checkout main
1. obtain the private key `openziti.rootCA.rsa.key`, sha256sum: e8de652fc6fb6b6189bb6de3162d3c3ea5ef019b1e80f6a94ed93f4bdd9d579f
1. cd to the root of the repo
1. create a new branch for the change you want to push 
1. issue the following command and optionally supply a valid `crl_reason`. [rfc5280, section 5.3.1](https://datatracker.ietf.org/doc/html/rfc5280#section-5.3.1)
   
        openssl ca \
            -config ./openziti.openssl.conf \
            -keyfile openziti.rootCA.rsa.key \
            -cert certs/openziti.rootCA.rsa.pem \
            -revoke certs/openziti.signing.2021.rsa.pem.torevoke \
            -crl_reason keyCompromise
   
1. sign the crl using the private key and generate the actual CRL 

       openssl ca -config ./openziti.openssl.conf -gencrl -out openziti.crl

1. commit and push the changes and issue PR to merge to main. Make sure to commit the following files:
    
        serial.num.txt
        certdb.txt
        openziti.crl

### Revocation Sanity Check

To verify the revoke command has succeeded you should notice a change to the certdb.txt file. The row you have revoked should
change from V for Valid, to R for Revoked. The second field of the file should now show a timestamp and the reason for which
the certificate was revoked. Example below:

    -V      220603050220Z           1004    unknown /CN=Code Signing Certificate .....
    +R      220603050220Z   210602113249Z,keyCompromise     1004    unknown /CN=Code Signing Certificate


## Testing A Revocation

The following commands were issued to generate a certificate which was then revoked:

    openssl genrsa -out openziti.signing.rsa.key.torevoke 4096

    openssl req \
        -new -key openziti.signing.rsa.key.torevoke \
        -config ./openziti.openssl.conf \
        -subj "/CN=RevokeTest Code Signing Certificate 2021/O=openziti.org Inc/OU=adv-dev/C=US/ST=NC" \
        -out openziti.signing.rsa.csr.torevoke
    
    openssl ca \
        -batch \
        -config ./openziti.openssl.conf \
        -keyfile openziti.rootCA.rsa.key \
        -cert certs/openziti.rootCA.rsa.pem \
        -days 1098 \
        -in openziti.signing.rsa.csr.torevoke \
        -extfile openziti.signing.rsa.conf \
        -out certs/openziti.signing.2021.rsa.pem.torevoke

    openssl ca \
        -config ./openziti.openssl.conf \
        -keyfile openziti.rootCA.rsa.key \
        -cert certs/openziti.rootCA.rsa.pem \
        -revoke certs/openziti.signing.2021.rsa.pem.torevoke \
        -crl_reason keyCompromise

    openssl pkcs12 \
        -export \
        -inkey openziti.signing.rsa.key.torevoke \
        -in certs/openziti.signing.2021.rsa.pem.torevoke \
        -out openziti.signing.rsa.pfx.torevoke