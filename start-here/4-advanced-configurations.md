# Advanced Configurations

You've deployed Coder, congratulations! The next steps

## Networking

### Certificate Creation

There's a variety of ways to generate certificates for your domain. By default, Coder doesn't enforce you to require TLS/SSL enabled, but depending on an organization's or your needs, you may want to create this to protect any HTTP traffic.

#### Certbot Installation

Probably the easiest way to create a certifcate is probably through `certbot`, a certificate authority generation and renewal tool. Depending on the operating system you're on, you may need to install certbot according to the OS's needs. The easiest way I've found to install certbot is through the Python PIP package manager.

Simply create and activte a virtual environment to isolate certbot into, install the certbot library through PIP, then run certbot!

```bash
$ mkdir certbot-env
$ cd certbot-env
$ python3 -m venv .venv
$ source .venv/bin/activate
(.venv) $ pip install certbot
```

There are a few things to note when generating a certificate. It's a good idea to review ["Let's Encrypt"](https://letsencrypt.org/how-it-works/) as this serves as a ["Certificate Authority"](https://en.wikipedia.org/wiki/Certificate_authority). Essentially, certbot by [default](https://eff-certbot.readthedocs.io/en/stable/man/certbot.html), points to "https://acme-v02.api.letsencrypt.org/directory", an ACME directory URL owned by Let's Encrypt. 

When first issuing a certificate for a domain, you first need to prove that you actually own it. A challenge will be issued to the requester, which will come in as a token to be embedded as a TXT record. You apply this TXT record to your domain, and then wait until Let's Encrypt can verify correctly that you are the actual owner.

This is a one time step, and won't happen again, even as you renew the certificate. You can follow the ["Let's Encrypt Getting Started"](https://letsencrypt.org/getting-started/) page for a pretty in-depth tutorial on how this works.

#### AWS - Route53

```bash
(.venv) $ pip install certbot-dns-route53
(.venv) $ certbot certonly --dns-route53 -d "domain1.com,*.domain1.com" --agree-tos --email myemail@gmail.com --eff-email
```

#### GCP - Google Cloud DNS

#### Azure - Azure DNS and Traffic Manager

### Applying Your Cert

Once you've acquired your certificate, it should now be located somewhere in your file system. With "let's encrypt", it'll be located in "/etc/letsencrypt/live/<YOUR-DOMAIN-NAME>". With any other certificate authority, be sure to follow their docs to see where a certificate and key gets stored.

After locating the desired files, you'll need to upload them to Coder. There's a few ways to do this, depending on what you're running:

#### K8s

With K8s, you can simply use the [CLI](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets) to generate a TLS secret. 

```bash
$ kubectl create secret tls my-tls-secret \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

Otherwise, if you wish to manually create it with a manifest, make sure to follow the format specified for "kubernetes.io/tls" Secret resource types.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  # All values are base64 encoded, which obscures them but does NOT provide any useful level of confidentiality
  tls.crt: ....
  # In this example, the key data is not a real PEM-encoded private key
  tls.key: ....   
```

#### Docker / Container Swarm

#### VM