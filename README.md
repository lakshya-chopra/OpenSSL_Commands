# List of a few handy OpenSSL commands:

1. Key generation:
```sh
openssl genpkey -algorithm ed25519 -out key.pem
```
2. Viewing this private key and getting the corresponding public key:
```sh
openssl pkey -in key.pem -text
openssl pkey -in privkey.pem -pubout -text
```
