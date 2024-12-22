# List of a few handy OpenSSL commands:

**1. Key generation:**
```sh
openssl genpkey -algorithm ed25519 -out key.pem
```
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
openssl req -x509 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 3650 -nodes

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

**8. Listing providers:**
```sh
openssl list -providers -verbose
```
Provider's capabilities:
```openssl list -providers -kem-algorithms -key-exchange-algorithms -signature-algorithms```

**9. s_client:**
Connecting to a remote host:
```sh
openssl s_client -connect google.com:443 -security_debug_verbose -msg -debug -state -status
```
Protocols can be specified as well, for example: `-tls1_1`,`-tls1_3`, `-dtls1_2`
