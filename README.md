# List of a few handy OpenSSL commands:

**1. Key generation:**
```sh
openssl genpkey -algorithm ed25519 -out key.pem
```
Extract pubkey:
```sh
openssl pkey -in key.pem -pubout -out pubkey.pem
```
DER to PEM:
```sh
openssl pkey -inform DER -in key.der -outform PEM -out key.pem
```
More
```sh
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out ec_private.pem
openssl ec -in ec_private.pem -pubout -out ec_public.pem
```
Read more pkeyopts [here](https://docs.openssl.org/3.4/man1/openssl-genpkey/#dsa-parameter-generation-options)

Other supported algorithms can be listed via: `openssl list -key-exchange-algorithms -signature-algorithms`

**2. Viewing the key:**
```sh
openssl pkey -in key.pem -text
```
Printing only the public key:
```sh
openssl pkey -in key.pem -text -pubout
```
Verifying the private-public keypair:
```sh
openssl pkey -in key.pem -text -pubout -check
```
`-text` option can be added at the end to print the key components in plaintext form, for example:
![image](https://github.com/user-attachments/assets/2b74520d-cb84-4b43-90ae-50c9d6e41df2)

`-noout` can be used to prevent openssl from printing the encoded form of the key.

**3. Generating a self-signed certificate in a single command:**
```sh
openssl req -x509 -newkey mldsa65 -keyout server.key -out server.crt -days 365 -nodes
```
`-nodes` can be used to prevent passphrase based encryption of the private key. Else ciphering algorithms such as `-aes256`, `chacha20` can be specified.

**4. Generating certificates via Private CA & CSRs**:
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
**5. Verifying the CSR**:
```sh
openssl req -in server.csr -noout -verify -text
```
**6.Verify whether the public key in the certificate matches with the corresponding private key:**
```sh
openssl x509 -in server.crt -pubkey -noout > pubkey.pem 
openssl pkey -in server.key -pubout -out privkey_pub.pem 
diff pubkey.pem privkey_pub.pem
```
**7. Random number generation:**
```sh
openssl rand -hex -num 32
```
Via some RNG engine, for example: `RDRAND`:
```sh
openssl rand -engine rdrand -hex -num 32
```
Other than `-hex`, `-base64` can also be used.

**8. Listing:**
- Providers
```sh
openssl list -providers -verbose
```
Provider's capabilities:
```openssl list -providers -kem-algorithms -key-exchange-algorithms -signature-algorithms```
- Algorithms:
```sh
openssl list -kem-algorithms -signature-algorithms -key-managers -public-key-algorithms -asymcipher-algorithms -key-exchange-algorithms -digest-algorithms -kdf-algorithms -mac-algorithms -cipher-algorithms
``` 

**9. s_client:**
Connecting to a remote host:
```sh
openssl s_client -connect google.com:443 -security_debug_verbose -msg -debug -state -status
```
Protocols can be specified as well, for example: `-tls1_1`,`-tls1_3`, `-dtls1_2`

**10. s_server:**
Starting a `DTLS 1.2` server
```sh
openssl s_server -cert srv.crt -key srv.key -dtls1_2
```
**11. Verify a Certificate Against a CA**
```sh
openssl verify -CAfile ca-cert.pem cert.pem
```

**12. View speed:**
Parallel:
```sh
openssl speed -seconds 5 -multi <cores> ed25519 ecdsa rsa3072 ed448
```
Single core:
```sh
openssl speed -seconds 5 ed25519 ecdsa rsa3072 ed448
```
