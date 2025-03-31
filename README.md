## Background
On creating a certificate through TrueNAS UI, it seems impossible to have SAN to include `IP Address:` as shown in the example below, which is inconvenient when the use case where the user accesses the TrueNAS using IP address directly exists.
So, here is the summary of steps to create a certificate that is signed by our own private CA created through TrueNAS UI, having SAN of `IP adress:`.


```sh
openssl x509 -in server.crt -noout -ext subjectAltName
```

```txt
X509v3 Subject Alternative Name:
    DNS:truenas.local, IP Address:192.168.101.110
```

## Steps
### Generate CSR (Certificate Signing Request)

#### Generate CSR and its private key separately (Option 1)
Generate the private key first.
Check if `server.key` is created in the current directory.

```sh
openssl genrsa -out server.key 4096
```

Then generate the CSR.
Check if `server.csr` is created in the current directory.

```sh
openssl req -new -key server.key -out server.csr -config csr_config.cnf
```

#### Generate CSR and its private key altogether (Option 2)
```sh
openssl req -new -config csr_config.cnf -out server.csr -newkey rsa -nodes -keyout server.key
```

Then check if `server.csr` and `server.key` are created in the current directory.
Note that the size of the private key is not specified but is instead read from csr_config.cnf file.


#### Verify the CSR
```sh
openssl req -in server.csr -noout -verify
```

Then check it prints a message like the following.

```txt
Certificate request self-signature verify OK
```

The content can also be checked as follows.

```sh
openssl req -in server.csr -noout -text
```


### Signing the CSR with pre-defined CA

#### Get CA certificate and its private key
Access TrueNAS UI and download them.
We assume that the certificate is named `ca.crt` and its private key is named `ca.key`.

#### Sign CSR using ca.crt and ca.key

```sh
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 825 -sha256 -extensions v3_req -extfile csr_config.cnf
```

Explanation:
* -req -in server.csr → Specifies the CSR to sign.
* -CA ca.crt -CAkey ca.key → Uses the CA certificate and its private key.
* -CAcreateserial → Generates a serial number file (ca.srl).
* -out server.crt → Outputs the signed certificate.
* -days 825 → Sets the certificate validity to 825 (maximum lifetime prior to Sep. 2020)
* -sha256 → Uses SHA-256 for better security.
* -extensions v3_req -extfile csr_config.cnf → Ensures the requested extensions (like SANs) are included.

#### Verify the certificate
```sh
openssl verify -CAfile ca.crt server.crt
```

Then check it prints a message like the following.

```txt
server.crt: OK
```

The content can also be checked as follows.

```sh
openssl x509 -in server.crt -noout -text
```
