# List of a few handy OpenSSL commands:

### Key operations
- Key generation
```sh
openssl genpkey -algorithm ed25519 -out key.pem
```
- Extract pubkey:
```sh
openssl pkey -in key.pem -pubout -out pubkey.pem
```
- DER to PEM:
```sh
openssl pkey -inform DER -in key.der -outform PEM -out key.pem
```
- Bonus:
```sh
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out ec_private.pem
openssl ec -in ec_private.pem -pubout -out ec_public.pem
```
```sh
openssl ecparam -list_curves   
openssl ecparam -name secp256k1 -text #View curve parameters
openssl ecparam -name prime256v1 -genkey -noout -out ec_private.pem
```
`ecparam` can also be used to generate Elliptic curve keys by specifiying curve parameters

- RSA specific:
```sh
openssl genpkey -algorithm RSA -aes-256-cbc -out rsa_private.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -in rsa_private.pem -pubout -out rsa_public.pem
openssl rsa -in rsa_private.pem -text
```
Sample:

![image](https://github.com/user-attachments/assets/1961f670-c1e2-40ab-939d-5476406cf5a8)

- **Example options**
  - RSA
  ```
  rsa_keygen_bits:4096
  rsa_keygen_pubexp:65537
  ```
> **Note:** OpenSSL rejects public exponents that fall outside a secure, allowed range to preserve cryptographic security.

##### Error demonstrated
```sh
openssl genpkey \
-algorithm RSA \
-out rsa2.pem \
-outform PEM \
-pkeyopt rsa_keygen_bits:2048 \
-pkeyopt rsa_keygen_pubexp:1024

genpkey: Error generating RSA key
4077A62F44790000:error:020000B2:rsa routines:rsa_multiprime_keygen:pub exponent out of range:../crypto/rsa/rsa_gen.c:96
  ```
  
  - RSA-PSS (`rsassaPss`)
  ```
  rsa_pss_keygen_md:sha256
  rsa_pss_keygen_mgf1_md:sha256
  rsa_pss_keygen_saltlen:32
  ```
  E.g.
  ```md
  openssl genpkey \
  -algorithm RSA-PSS \
  -out rsa.pem \
  -outform PEM \
  -pkeyopt rsa_keygen_bits:4096 \
  -pkeyopt rsa_pss_keygen_md:sha256
  ```
  - ECDSA
  ```md
  ec_paramgen_curve:<curve_name>
  ```

> Note: Other supported algorithms can be listed via: `openssl list -key-exchange-algorithms -signature-algorithms`

- Viewing the key:
```sh
openssl pkey -in key.pem -text
```
Example:
```sh
openssl pkey -in rsa.pem -text -noout
```
- ASN1 Parse:
```sh
openssl asn1parse -in rsa.pem -i
```
- Printing only the public key:
```sh
openssl pkey -in key.pem -text -pubout
```
- Verifying the private-public keypair:
```sh
openssl pkey -in key.pem -text -pubout -check
```
`-text` option can be added at the end to print the key components in plaintext form, for example:
![image](https://github.com/user-attachments/assets/2b74520d-cb84-4b43-90ae-50c9d6e41df2)

`-noout` can be used to prevent openssl from printing the encoded form of the key.

### Generating a self-signed certificate in a single command
```sh
openssl req -x509 -newkey mldsa65 -keyout server.key -out server.crt -days 365 -nodes
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=localhost"
```
`-nodes` can be used to prevent passphrase based encryption of the private key. Else ciphering algorithms such as `-aes256`, `chacha20` can be specified.

### Generating certificates via Private CA & CSRs
```sh 
# Generating CA 
openssl req -x509 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 3650 -subj "/CN=test CA" -nodes

# Viewing CA cert
openssl x509 -in ca.crt -text -noout 

# Generating RSA key
openssl genpkey -algorithm RSA -out server.key

# CSR gen
openssl req -new -key server.key -out server.csr

# Signing CSR
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
```
### Verifying the CSR
```sh
openssl req -in server.csr -noout -verify -text
```
### Verify whether the public key in the certificate matches with the corresponding private key
```sh
openssl x509 -in server.crt -pubkey -noout > pubkey.pem 
openssl pkey -in server.key -pubout -out privkey_pub.pem 
diff pubkey.pem privkey_pub.pem
```
### Random number generation
```sh
openssl rand -hex -num 32
```
Via some RNG engine, for example: `RDRAND`:
```sh
openssl rand -engine rdrand -hex -num 32
```
Other than `-hex`, `-base64` can also be used.

### Listing
- Providers
```sh
openssl list -providers -verbose
```
Provider's capabilities:
```openssl list -providers -kem-algorithms -key-exchange-algorithms -signature-algorithms```
- Algorithms:
```sh
openssl list -kem-algorithms -signature-algorithms -key-managers -public-key-algorithms \
-asymcipher-algorithms -key-exchange-algorithms -digest-algorithms -kdf-algorithms \
-mac-algorithms -cipher-algorithms
``` 

### TLS connection: s_client and s_server
- Connecting to a remote host:
```sh
openssl s_client -connect google.com:443 -security_debug_verbose -msg -debug -state -status
```
Protocols can be specified as well, for example: `-tls1_1`,`-tls1_3`, `-dtls1_2`

- Starting a `DTLS 1.2` server
```sh
openssl s_server -cert srv.crt -key srv.key -dtls1_2
```
### PKI

- Verify a Certificate Against a CA**
```sh
openssl verify -CAfile ca-cert.pem cert.pem
```
- Without:
```sh
openssl verify -CAfile ca.crt server.crt
```

### Performance
- Parallel:
```sh
openssl speed -seconds 5 -multi <cores> ed25519 ecdsa rsa3072 ed448
```
-- Single core:
```sh
openssl speed -seconds 5 ed25519 ecdsa rsa3072 ed448
```

## References:
- pkeyopts [here](https://docs.openssl.org/3.4/man1/openssl-genpkey/#dsa-parameter-generation-options)
