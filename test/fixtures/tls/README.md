Test certificates for `test/tls_test.qz`, valid until October 2076 (not
secret: the key is for these tests only).

- `ca.pem`: the test CA (P-256), whose key was discarded.
- `server.pem`, `server.key`: the server's certificate for
  `DNS:localhost, IP:127.0.0.1, IP:::1`, signed by the CA, and its key.
  The tests give their contents to `PG::Testing.start(certificate:,
  private_key:)`, which writes them into the cluster's directory (the
  key file mode 0600, as the server wants).
- `other-ca.pem`: an unrelated CA, for the unknown-issuer test.

Made with OpenSSL 3.5:

```sh
W=$(mktemp -d)
openssl ecparam -name prime256v1 -genkey -noout -out $W/ca.key
openssl req -x509 -new -key $W/ca.key -sha256 -days 18250 -subj "/CN=pg test CA" \
  -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign" -out ca.pem
openssl ecparam -name prime256v1 -genkey -noout -out $W/other.key
openssl req -x509 -new -key $W/other.key -sha256 -days 18250 -subj "/CN=pg other CA" \
  -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign" -out other-ca.pem
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out server.key
openssl req -new -key server.key -subj "/CN=localhost" -out $W/server.csr
printf 'subjectAltName=DNS:localhost,IP:127.0.0.1,IP:::1\nbasicConstraints=critical,CA:FALSE\nkeyUsage=critical,digitalSignature\nextendedKeyUsage=serverAuth\n' > $W/ext
openssl x509 -req -in $W/server.csr -CA ca.pem -CAkey $W/ca.key -CAcreateserial -CAserial $W/ca.srl \
  -days 18250 -sha256 -extfile $W/ext -out server.pem
rm -rf $W
```
