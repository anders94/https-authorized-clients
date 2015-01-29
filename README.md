HTTPS Authorized Certs
======================

Typically, HTTPS servers do a basic TLS handshake and accept any client connection as 
long as a compatible cipher suite can be found. However, the server can be configured 
to send the client a CertificateRequest during the TLS handshake which requires the
client to present a certificate as a form of identity.

* [http://en.wikipedia.org/wiki/Transport_Layer_Security#Client-authenticated_TLS_handshake](Client-authenticated TLS Handshakes) (from Wikipedia)

HTTPS server certificates usually have their "Common Name" set to their fully qualified 
domain name and are signed by a well known certificate authority such as Verisign. 
However, the "Common Name" usually used in client certificates can be set to anything that
identifies the client such as "Acme, Co." or "client-12345". This will be presented to the 
server and can be used in addition to or instead of username / password strategies to
identify the client.

Using node.js, one can instruct the server to request a client certificate and reject 
unauthorized clients by adding

    {
      requestCert: true,
      rejectUnauthorized: true
    }

to the options passed to https.createServer(). In turn, a client will be rejected unless
it passes a valid certificate in its https.request() options.

    {
      key: fs.readFileSync('keys/client-key.pem'),
      cert: fs.readFileSync('keys/client-crt.pem')
    }

The following exercise will create a self signed certificate authority, server certificate and 
two client certificates all "self signed" by the certificate authority. Then we will run an 
HTTPS server which will accept only connections made by clients presenting a valid certificate.
We will finish off by revoking one of the client certificates and seeing that the server 
rejects requests from this client.

Setup
=====

Let's create our own certificate authority so we can sign our own client certificates. We will
also sign our server certificate so we don't have to pay for one for our server.

Create a Certificate Authority
------------------------------

We will do this only once and use the configuration stored in keys/ca.cnf. A 27 year certificate 
(9999 days) of 4096 bits should do the trick quite well. (we want our CA to be valid for a long
time and be super secure - but this is really just overkill)

    openssl req -new -x509 -days 9999 -config keys/ca.cnf -keyout keys/ca-key.pem -out keys/ca-crt.pem

Now we have a certificate authority with the private key keys/ca-key.pem and the public key 
keys/ca-crt.pem.

Create Private Keys
-------------------

Let's build some private keys for our server and client certificates.

    openssl genrsa -out keys/server-key.pem 4096
    openssl genrsa -out keys/client1-key.pem 4096
    openssl genrsa -out keys/client2-key.pem 4096

Again, 4096 is a bit of overkill here but we aren't too worried about CPU usage issues.

Sign Certificates
-----------------

Now let's sign these certificates using the certificate authority we made previously. This is usually
called "self signing" our certificates. We'll start by signing the server certificate.

    openssl req -new -config keys/server.cnf -key keys/server-key.pem -out keys/server-csr.pem
    openssl x509 -req -extfile keys/server.cnf -days 999 -passin "pass:password" -in keys/server-csr.pem -CA keys/ca-crt.pem -CAkey keys/ca-key.pem -CAcreateserial -out keys/server-crt.pem

The first line creates a "CSR" or certificate signing request which is written to keys/server-csr.pem
Next we use the configuration stored in keys/server.cnf and our certificate authority to sign the CSR
resulting in keys/server-crt.pem, our server's new public certificate.

Let's do the same for the two client certificates, using different configuration files. (the configuration 
files are identical except for the Common Name setting so we can distinguish them later)

    openssl req -new -config keys/client1.cnf -key keys/client1-key.pem -out keys/client1-csr.pem
    openssl x509 -req -extfile keys/client1.cnf -days 999 -passin "pass:password" -in keys/client1-csr.pem -CA keys/ca-crt.pem -CAkey keys/ca-key.pem -CAcreateserial -out keys/client1-crt.pem

    openssl req -new -config keys/client2.cnf -key keys/client2-key.pem -out keys/client2-csr.pem
    openssl x509 -req -extfile keys/client2.cnf -days 999 -passin "pass:password" -in keys/client2-csr.pem -CA keys/ca-crt.pem -CAkey keys/ca-key.pem -CAcreateserial -out keys/client2-crt.pem

OK, we should be set with the certificates we need.

Verify
------

Let's just test them out though to make sure each of these certificates has been validly signed by our
certificate authority.

    openssl verify -CAfile keys/ca-crt.pem keys/server-crt.pem
    openssl verify -CAfile keys/ca-crt.pem keys/client1-crt.pem
    openssl verify -CAfile keys/ca-crt.pem keys/client2-crt.pem

If we get an "OK" when running each of those commands, we are all set.

Run the Example
===============

We should be ready to go now. Let's fire up the server:

    node server

We now have a server listening on 0.0.0.0:4433 that will only work if the client presents a valid 
certificate signed by the certificate authority. Let's test that out in another window:

    node client 1

This will invoke a client using the client1-crt.pem certificate which should connect to the server
and get a "hello world" back in the body. Let's try it with the other client certificate as well:

    node client 2

You should be able to see from the server output that it can distinguish between the two clients
by the certificates they present. (client1 or client2 which are the Common Names set in the .cnf 
files)

Certificate Revocation
======================

All is well in the world until we want to shut down a specific client without shutting everybody 
else down and regenerating certificates. Let's create a Certificate Revocation List (CRL) and 
revoke the client2 certificate. The first time we'll do this, we need to create an empty database:

    touch ca-database.txt

Now let's revoke client2's certificate and update the CRL:

    openssl ca -revoke keys/client2-crt.pem -keyfile keys/ca-key.pem -config keys/ca.cnf -cert keys/ca-crt.pem -passin 'pass:password'
    openssl ca -keyfile keys/ca-key.pem -cert keys/ca-crt.pem -config keys/ca.cnf -gencrl -out keys/ca-crl.pem -passin 'pass:password'

Let's stop the server and comment back in line 8 which reads in the CRL:

    crl: fs.readFileSync('keys/ca-crl.pem')

and restart the server again:

    node server

Now comes the moment of truth. Let's test to see if client 2 works or not:

    node client 2

If all goes well, it won't work anymore. Just as a sanity check, let's make sure client 1 still works:

    node client 1

Likewise, if all is well, client 1 still works while client 2 is rejected.

Conclusion
==========

We have seen how we can create self signed server and client certificates and ensure that clients
interacting with our server only use valid certificates signed by us. Additionally, we can revoke 
any of the client certificates without having to revoke everything and rebuild from scratch. 
Because we can see the Common Name of the client certificates being presented and we know that they
must be valid in order for us to see them, we can use this as a strategy to identify clients using
our server.

Author
======
**Anders Brownworth**

+ [http://twitter.com/anders94](@anders94)
+ [http://github.com/anders94](github.com/anders94)
+ [http://anders.com/](anders.com)
