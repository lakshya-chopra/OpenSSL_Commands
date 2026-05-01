# OpenSSL
---

## Key Operations

### Generation

```sh
openssl genpkey -algorithm ed25519 -out key.pem
```

### Extract public key

```sh
openssl pkey -in key.pem -pubout -out pubkey.pem
```

### DER to PEM

```sh
openssl pkey -inform DER -in key.der -outform PEM -out key.pem
```

---

### EC Keys

```sh
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out ec_private.pem
openssl ec -in ec_private.pem -pubout -out ec_public.pem
```

```sh
openssl ecparam -list_curves
openssl ecparam -name secp256k1 -text          # View curve parameters
openssl ecparam -name prime256v1 -genkey -noout -out ec_private.pem
```

> `ecparam` can also generate EC keys directly by specifying curve parameters.

---

### RSA Keys

```sh
openssl genpkey -algorithm RSA -aes-256-cbc -out rsa_private.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -in rsa_private.pem -pubout -out rsa_public.pem
openssl rsa -in rsa_private.pem -text
```

**`-pkeyopt` options by algorithm:**

- **RSA**
  ```
  rsa_keygen_bits:4096
  rsa_keygen_pubexp:65537
  ```

  > **Note:** OpenSSL rejects public exponents outside a secure allowed range.

  Example error when using an out-of-range exponent:
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

- **RSA-PSS** (`rsassaPss`)
  ```
  rsa_pss_keygen_md:sha256
  rsa_pss_keygen_mgf1_md:sha256
  rsa_pss_keygen_saltlen:32
  ```

  Example:
  ```sh
  openssl genpkey \
    -algorithm RSA-PSS \
    -out rsa.pem \
    -outform PEM \
    -pkeyopt rsa_keygen_bits:4096 \
    -pkeyopt rsa_pss_keygen_md:sha256
  ```

- **ECDSA**
  ```
  ec_paramgen_curve:<curve_name>
  ```

> Other supported algorithms: `openssl list -key-exchange-algorithms -signature-algorithms`

---

### Viewing & Inspecting Keys

```sh
openssl pkey -in key.pem -text                  # View key
openssl pkey -in rsa.pem -text -noout           # View key, suppress encoded output
openssl pkey -in key.pem -text -pubout          # Print only public key
openssl pkey -in key.pem -text -pubout -check   # Verify private-public keypair
openssl asn1parse -in rsa.pem -i                # ASN1 parse
```

> `-noout` suppresses the PEM-encoded output; `-text` prints key components in plaintext.

---

## Certificates

### Self-Signed (single command)

```sh
openssl req -x509 -newkey mldsa65 -keyout server.key -out server.crt -days 365 -nodes
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=localhost"
```

> `-nodes` skips passphrase encryption of the private key. Alternatively, specify a cipher like `-aes256` or `-chacha20`.

---

### Via Private CA & CSR

```sh
# Generate CA
openssl req -x509 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 3650 -subj "/CN=test CA" -nodes

# View CA cert
openssl x509 -in ca.crt -text -noout

# Generate server key
openssl genpkey -algorithm RSA -out server.key

# Generate CSR
openssl req -new -key server.key -out server.csr

# Sign CSR with CA
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
```

### Verify CSR

```sh
openssl req -in server.csr -noout -verify -text
```

### Check cert ↔ private key match

```sh
openssl x509 -in server.crt -pubkey -noout > pubkey.pem
openssl pkey -in server.key -pubout -out privkey_pub.pem
diff pubkey.pem privkey_pub.pem
```

---

## Random Number Generation

```sh
openssl rand -hex -num 32
openssl rand -base64 32
```

Via a specific RNG engine (e.g. `RDRAND`):

```sh
openssl rand -engine rdrand -hex -num 32
```

---

## Listing Providers & Algorithms

```sh
# Providers
openssl list -providers -verbose
openssl list -providers -kem-algorithms -key-exchange-algorithms -signature-algorithms

# Algorithms
openssl list \
  -kem-algorithms -signature-algorithms -key-managers -public-key-algorithms \
  -asymcipher-algorithms -key-exchange-algorithms -digest-algorithms -kdf-algorithms \
  -mac-algorithms -cipher-algorithms
```

---

## TLS — `s_client` & `s_server`

```sh
# Connect to remote host
openssl s_client -connect google.com:443 -security_debug_verbose -msg -debug -state -status

# Start a DTLS 1.2 server
openssl s_server -cert srv.crt -key srv.key -dtls1_2
```

> Protocols can be specified with flags like `-tls1_1`, `-tls1_3`, `-dtls1_2`.

---

### The RSA-PSS / TLS 1.3 Key Duality

Many certificates issued today use `sha256WithRSAEncryption`, which is **RSASSA-PKCS#1 v1.5**, which is an algorithm explicitly forbidden in TLS 1.3 for signing handshake transcripts. To work around this, a server holding such a certificate will use its underlying RSA key to sign the transcript with `rsa_pss_rsae_sha256` instead. The key effectively plays dual roles.

This distinction becomes clearer when you look at three scenarios:

---

#### Case 1 — RSA key with a PKCS#1 v1.5 certificate

1. Generate the keypair and certificate:
   ```sh
   openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out rsa.key

   openssl req -new -x509 -key rsa.key -out rsa_v15.crt \
     -days 365 -subj "/CN=example.com" -sha256
   ```

2. Check the algorithms:
   ```sh
   openssl x509 -in rsa_v15.crt -text -noout | grep -E "Signature Algorithm|Public Key Algorithm" | head -2
   ```
   ```
   Public Key Algorithm: rsaEncryption
   Signature Algorithm:  sha256WithRSAEncryption
   ```

3. Start a server and connect a TLS 1.3 client, requesting `rsa_pss_rsae_sha256`:
   ```sh
   openssl s_server -cert rsa_v15.crt -key rsa.key -tls1_3
   ```
   ```sh
   openssl s_client -connect localhost:4433 -tls1_3 -sigalgs rsa_pss_rsae_sha256
   ```

   Despite the certificate using PKCS#1 v1.5, the handshake succeeds, the key is used to sign with PSS. The client reports:
   ```
   Peer signature type: RSA-PSS
   ```
   And in the certificate chain:
   ```
   PKEY: rsaEncryption
   ```

---

#### Case 2 — RSA key with a PSS-signed certificate (unrestricted)

Same as Case 1, except the certificate is signed with PSS padding:

```sh
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out rsa_pss.key

openssl req -new -x509 -key rsa_pss.key -out rsa_pss_cert.crt \
  -days 365 -subj "/CN=example.com" \
  -sigopt rsa_padding_mode:pss \
  -sigopt rsa_pss_saltlen:-1 \
  -sha256
```

The `Signature Algorithm` will differ, but the TLS handshake results (peer signature type, PKEY) are identical to Case 1: the key type is still `rsaEncryption` and both `rsa_pss_rsae_sha256` and `rsa_pss_pss_sha256` work.

---

#### Case 3 — RSA-PSS key (restricted)

Here the key itself is of type `rsassaPss`, meaning it is restricted to PSS-only usage. Using `rsa_pss_rsae_sha256` (which expects an `rsaEncryption` key) will fail; only `rsa_pss_pss_sha256` succeeds.

1. Generate the restricted key and certificate:
   ```sh
   openssl genpkey -algorithm RSA-PSS \
     -pkeyopt rsa_keygen_bits:2048 \
     -pkeyopt rsa_pss_keygen_md:sha256 \
     -pkeyopt rsa_pss_keygen_mgf1_md:sha256 \
     -pkeyopt rsa_pss_keygen_saltlen:32 \
     -out rsapss.key

   openssl req -new -x509 -key rsapss.key -out rsapss_cert.crt \
     -days 365 -subj "/CN=example.com" \
     -sigopt rsa_padding_mode:pss \
     -sigopt rsa_pss_saltlen:-1 \
     -sha256
   ```

2. Verify algorithms:
   ```
   Public Key Algorithm: rsassaPss
   Signature Algorithm:  rsassaPss
   ```

3. Test all three `sigalgs` options:
   - `rsa_pss_pss_sha256` — succeeds
   - `rsa_pss_rsae_sha256` — fails
   - *(none specified)* — fails

---

## PKI — Certificate Verification

```sh
openssl verify -CAfile ca-cert.pem cert.pem
openssl verify -CAfile ca.crt server.crt
```

---

## Performance Benchmarks

```sh
# Multi-core
openssl speed -seconds 5 -multi <cores> ed25519 ecdsa rsa3072 ed448

# Single core
openssl speed -seconds 5 ed25519 ecdsa rsa3072 ed448
```

---

## References

- `genpkey` pkeyopts: https://docs.openssl.org/3.4/man1/openssl-genpkey/#dsa-parameter-generation-options
