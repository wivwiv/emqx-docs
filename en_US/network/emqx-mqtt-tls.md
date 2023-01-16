# Enable SSL/TLS

SSL/TLS is an industry-standard protocol for encrypting network communications at the transport layer to improve data security and ensure data integrity. 

EMQX fully supports SSL/TLS capabilities, including support for one-way/two-way authentication and X.509 certificate authentication. All EMQX connections, including MQTT clients, can be encrypted with SSL/TLS to ensure safety.

This chapter will introduce the advantages of SSL/TLS encrypted connections and how to enable SSL/TLS connections in EMQX.

## Advantages

1. Strong authentication: With TLS connection enabled, different communication parties can confirm each other's identity, for example, by checking their X.509 digital certificate. This digital certificate is usually issued by a trusted organization Certificate Authority (CA) and cannot be forged.

2. Confidentiality: With TLS connection enabled, every session will be encrypted according to the session key negotiated by both parties. No third party can know the communication content, so even if the key of one session is leaked, it will not affect the security of other sessions.

3. Integrity: The possibility of data tampering  in encrypted communications is extremely low.

   

For SSL/TLS connection in EMQX, you can enable a direct connection between EMQX and the clients or terminate SSL/TLS  connections with an agent or load balancer.

| Usage                                                        | Advantage                                                    | Disadvantage                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Enable a direct connection between EMQX and the clients      | Simple and easy to use, and no additional components are required. | Resouce-consuming, especially when the number of connections is high, it may consume a large amount of CPU and memory resources. |
| Terminate SSL/TLS  connections with an agent or load balancer | No impact on EMQX performance while providing the load balancing capability. | Only a few cloud vendors provide load balancers that  support TCP SSL/TLS termination. And users need to deploy some software before using this feature, such as [HAProxy](http://www.haproxy.org/). |

This chapter will focus on how to enable SSL/TLS connection between the client and EMQX. For how to terminate TLS connections through a proxy or load balancer, please refer to [Cluster Load Balancing](../deploy/cluster/lb.md).

## Enable SSL/TLS connection with configuration files

:::tip

Prerequisites: SSL/TLS certificates are ready. 

EMQX provides a set of SSL/TLS certificates (in the `etc/certs` directory) with the installation package and enables SSL/TLS connections on port `8883`. The attached SSL/TLS certificates are for testing and verification only. You should switch to certificates signed by a trusted CA for the production environment.

On how to apply for SSL/TLS certificates, see [Further reading: How to obtain SSL/TLS certificates](#Further-reading:-How-to-obtain-SSL/TLS-certificates).

:::

1. Move the SSL/TLS certificates to the EMQX `etc/cert` directory.
2. Open the configuration file `emqx.conf` (in the `./etc` or `/etc/emqx/etc` directory based on your installation mode), modify the `listeners.ssl.default` configuration group, replace the certificate with your certificate, and add `verify=verify_none`:

```bash
listeners.ssl.default {
  bind = "0.0.0.0:8883"
  max_connections = 512000
  ssl_options {
    # keyfile = "etc/certs/key.pem"
    keyfile = "etc/certs/server.key"
    # certfile = "etc/certs/cert.pem"
    certfile = "etc/certs/server.crt"
    # cacertfile = "etc/certs/cacert.pem"
    cacertfile = "etc/certs/rootCA.crt"

    # Do not enable peer verification
    verify = verify_none

  }
}
```

So far, we have completed the SSL/TLS one-way authentication configuration on EMQX, which will ensure the communications are encrypted but cannot verify the identity the client's identity..

To enable a two-way authentication, please add the following configuration in `listeners.ssl.default`:

```bash
listeners.ssl.default {
  ...
  ssl_options {
    ...
    # Enable peer verification
    verify = verify_peer
    # Force two-way authentication to be enabled, if the client cannot provide a certificate, the connection will be rejected
    fail_if_no_peer_cert = true
  }
}
```

3. Restart EMQX to apply the above configuration.

## Further reading: How to obtain SSL/TLS certificates

You can obtain SSL/TLS certificates in either of the following ways:

1. Self-signed certificates: Since self-signed certificates have many security risks, it is only recommended for testing and verification.
2. Apply for or purchase SSL/TLS certificates: You can apply for certificates from institutions like [Let's Encrypt](https://letsencrypt.org/) or cloud vendors like AWS, Microsoft Azure, and Google Cloud, or purchase paid certificates from institutions like [DigiCert](https://www.digicert.com/). For enterprise users, it is recommended to use paid Organization Validation (OV) certificates for better security protection.

### Create self-signed certificates

:::tip

Prerequisites: [OpenSSL](https://www.openssl.org/) is installed. 

:::

1. Run the following command to generate a key pair (`rootCA.key`). The command will prompt you to enter the key protection password, which will be required when generating, issuing, and verifying certificates. Please keep your key and password properly.

  ```bash
openssl genrsa -des3 -out rootCA.key 2048
  ```

2. Run the following command to generate a CA certificate (`rootCA.crt`) from the private key in `rootCA.key`. The command will prompt you to set the certificate's Distinguished Name (DN).

  ```bash
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.crt
  ```

3. Use the `rootCA.crt` generated in step 2 to issue a server certificate to verify the identity of the server owner. The server certificate is usually issued to the hostname, server name or domain name (such as www.emqx.com). We need CA key (`rootCA.key`), CA certificate (`rootCA.crt`) and server CSR (`server.csr`) to generate the server certificate.

   3.1 Run the command below to generate the `server.key`:


  		```bash
  		openssl genrsa -out server.key 2048
  		```

​		3.2 Run the command below to make the server CSR (`server.csr`) with `server.key`. After being signed by the private key of the `CA root certificate`, the CSR can generate the public key file issued to the user. The command will then prompt you to set the certificate's DN.

  		```bash
  		openssl req -new -key server.key -out server.csr
  		 ```

 		Below information will be returned:

  ```bash
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]: 
State or Province Name (full name) [Some-State]: 
Locality Name (eg, city) []: 
Organization Name (eg, company) [Internet Widgits Pty Ltd]: # such as EMQ
Organizational Unit Name (eg, section) []: # such as EMQX
Common Name (e.g. server FQDN or YOUR name) []: # domain name of the server, for example mqtt.emqx.com
...
  ```

​		3.3 Generate a server certificate (`server.crt`). You can also specify the validity period of the certificate, for example, 365 days:

  ```bash
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out server.crt -days 365
  ```

At this point you have a set of certificates.

```bash
.
├── rootCA.crt
├── rootCA.key
├── rootCA.srl
├── server.crt
├── server.csr
└── server.key
```

<!--Apply for or purchase certificates -->
