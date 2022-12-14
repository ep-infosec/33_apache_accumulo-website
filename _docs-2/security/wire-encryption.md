---
title: Wire Encryption
category: security
order: 6
---

Accumulo, through Thrift's TSSLTransport, provides the ability to encrypt
wire communication between Accumulo servers and clients using secure
sockets layer (SSL). SSL certificates signed by the same certificate authority
control the "circle of trust" in which a secure connection can be established.
Typically, each host running Accumulo processes would be given a certificate
which identifies itself.

Clients can optionally also be given a certificate, when client-auth is enabled,
which prevents unwanted clients from accessing the system. The SSL integration
presently provides no authentication support within Accumulo (an Accumulo username
and password are still required) and is only used to establish a means for
secure communication.

## Server configuration

As previously mentioned, the circle of trust is established by the certificate
authority which created the certificates in use. Because of the tight coupling
of certificate generation with an organization's policies, Accumulo does not
provide a method in which to automatically create the necessary SSL components.

Administrators without existing infrastructure built on SSL are encourage to
use OpenSSL and the `keytool` command. An example of these commands are
included in a section below. Accumulo servers require a certificate and keystore,
in the form of Java KeyStores, to enable SSL. The following configuration assumes
these files already exist.

In `accumulo.properties`, the following properties are required:

* {% plink rpc.javax.net.ssl.keyStore %}  = _The path on the local filesystem to the keystore containing the server's certificate_
* {% plink rpc.javax.net.ssl.keyStorePassword %} = _The password for the keystore containing the server's certificate_
* {% plink rpc.javax.net.ssl.trustStore %} = _The path on the local filesystem to the keystore containing the certificate authority's public key_
* {% plink rpc.javax.net.ssl.trustStorePassword %} = _The password for the keystore containing the certificate authority's public key_
* {% plink instance.rpc.ssl.enabled %} = _true_

Optionally, SSL client-authentication (two-way SSL) can also be enabled by setting
{% plink instance.rpc.ssl.clientAuth %} `true` in `accumulo.properties`.
This requires that each client has access to a valid certificate to set up a secure connection
to the servers. By default, Accumulo uses one-way SSL which does not require clients to have
their own certificate.

## Client configuration

To establish a connection to Accumulo servers, each client must also have
special configuration. This is typically accomplished by [creating an Accumulo
client][clients] using `accumulo-client.properties` and setting the following
the properties to connect to an Accumulo instance using SSL:

* {% plink -c ssl.enabled %} to `true`
* {% plink -c ssl.truststore.path %}
* {% plink -c ssl.truststore.password %}

If two-way SSL is enabled for the Accumulo instance (by setting {% plink instance.rpc.ssl.clientAuth %} to `true` in `accumulo.properties`),
Accumulo clients must also define their own certificate by setting the following properties:

* {% plink -c ssl.keystore.path %}
* {% plink -c ssl.keystore.password %}

## Generating SSL material using OpenSSL

The following is included as an example for generating your own SSL material (certificate authority and server/client
certificates) using OpenSSL and Java's KeyTool command.

### Generate a certificate authority

```shell
# Create a private key
openssl genrsa -des3 -out root.key 4096

# Create a certificate request using the private key
openssl req -x509 -new -key root.key -days 365 -out root.pem

# Generate a Base64-encoded version of the PEM just created
openssl x509 -outform der -in root.pem -out root.der

# Import the key into a Java KeyStore
keytool -import -alias root-key -keystore truststore.jks -file root.der

# Remove the DER formatted key file (as we don't need it anymore)
rm root.der
```

The `truststore.jks` file is the Java keystore which contains the certificate authority's public key.

### Generate a certificate/keystore per host

It's common that each host in the instance is issued its own certificate (notably to ensure that revocation procedures
can be easily followed). The following steps can be taken for each host.

```shell
# Create the private key for our server
openssl genrsa -out server.key 4096

# Generate a certificate signing request (CSR) with our private key
openssl req -new -key server.key -out server.csr

# Use the CSR and the CA to create a certificate for the server (a reply to the CSR)
openssl x509 -req -in server.csr -CA root.pem -CAkey root.key -CAcreateserial \
    -out server.crt -days 365

# Use the certificate and the private key for our server to create PKCS12 file
openssl pkcs12 -export -in server.crt -inkey server.key -certfile server.crt \
    -name 'server-key' -out server.p12

# Create a Java KeyStore for the server using the PKCS12 file (private key)
keytool -importkeystore -srckeystore server.p12 -srcstoretype pkcs12 -destkeystore \
    server.jks -deststoretype JKS

# Remove the PKCS12 file as we don't need it
rm server.p12

# Import the CA-signed certificate to the keystore
keytool -import -trustcacerts -alias server-crt -file server.crt -keystore server.jks
```

The `server.jks` file is the Java keystore containing the certificate for a given host. The above
methods are equivalent whether the certificate is generated for an Accumulo server or a client.

[clients]: {{ page.docs_baseurl }}/getting-started/clients#creating-an-accumulo-client
