# Traefik step-ca local ACME PKI

[Читать на русском](./README.ru.md)

## Description

This repository demonstrates how to configure Traefik to work with step-ca as a certificate authority (CA).  
Goal: obtain and use certificates signed by your own CA (Root + Intermediate) instead of Let’s Encrypt.

## Background

Originally, Traefik was used with a wildcard certificate in environments where Let’s Encrypt certificates were not feasible.  
This can be done as follows:

In `traefik.yml`, enable the file provider:
```yml
file:
  directory: /custom
  watch: true
```
Then in `traefik/data/custom/**dynamic.yml**`:
```yml
http:
  routers:
    traefik:
      rule: "Host(`traefik.domain.example.com`)"
      service: "api@internal"
      tls:
        domains:
          - main: "*.${DOMAIN}"
tls:
  certificates:
    - certFile: "/ssl/cert.pem"
      keyFile: "/ssl/privkey.pem"
```
Environment variables in `.env`:
```bash
DOMAIN=home.arpa
DOMAIN_API=traefik-api.home.arpa
```
This way you can use a wildcard certificate with Traefik.  
But it is much more interesting to run your own CA and obtain certificates there, which is described below.

# Setup Steps

- You need the ability to manage DNS A records.
- Docker and Docker Compose plugin must be installed.
- Root and intermediate certificates are required.
> Any tool can be used to generate certificates: easy-rsa, openssl, etc.

## smallstep/step-ca

Step-ca can be run following their [official Docker instructions](https://hub.docker.com/r/smallstep/step-ca) or [here](https://smallstep.com/docs/tutorials/docker-tls-certificate-authority/index.html).

### Docker-compose example
```yml
services:
  step-ca:
    image: 'smallstep/step-ca:latest'
    container_name: 'acme-ca'
    hostname: 'acme-ca'
    ports:
      - 9000:9000
    dns:
      - '${DNS1}'
      - '${DNS2}'
    dns_search:
      - '${DOMAIN}'
    environment:
      - "DOCKER_STEPCA_INIT_NAME=${CA_NAME}"
      - "DOCKER_STEPCA_INIT_DNS_NAMES=${CA_DNS_NAMES}"
      - "DOCKER_STEPCA_INIT_PROVISIONER_NAME=${STEP_PROVISIONER_NAME}"
      - "DOCKER_STEPCA_INIT_PASSWORD=${CA_ENCRYPTION_PASS}"
      - "DOCKER_STEPCA_INIT_SSH=${INIT_SSH}"
    volumes:
      - './data:/home/step'
      - '/etc/localtime:/etc/localtime:ro'
```

### Environment Variables (.env)
```bash
CA_NAME="Smalstep intermediate ACME CA"
CA_DNS_NAMES=acme-ca.home.arpa
STEP_PROVISIONER_NAME=admin
CA_ENCRYPTION_PASS="<somegeneratedpass>"
INIT_SSH=
DOMAIN=home.arpa
DNS1=192.168.1.250
DNS2=192.168.1.240
```

### Step-ca Configuration (`ca.json`)
```json
{
    "root": "/home/step/certs/root-ca.crt",
    "federatedRoots": null,
    "crt": "/home/step/certs/intermediate-ca.crt",
    "key": "/home/step/secrets/intermediate-ca.key",
    "address": ":9000",
    "insecureAddress": "",
    "dnsNames": ["localhost", "acme-ca.home.arpa"],
    "logger": {"format": "text"},
    "db": {"type": "badgerv2", "dataSource": "/home/step/db"},
    "authority": {
        "provisioners": [{
            "type": "ACME",
            "name": "acme",
            "forceCN": true,
            "claims": {
                "enableSSHCA": true,
                "disableRenewal": false,
                "allowRenewalAfterExpiry": false,
                "disableSmallstepExtensions": false,
                "maxTLSCertDuration": "2160h",
                "defaultTLSCertDuration": "2160h"
            },
            "options": {"x509": {}, "ssh": {}}
        }],
        "template": {},
        "backdate": "1m0s",
        "enableAdmin": true
    },
    "tls": {
        "cipherSuites": [
            "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256",
            "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
        ],
        "minVersion": 1.2,
        "maxVersion": 1.3,
        "renegotiation": false
    },
    "commonName": "acme-ca.home.arpa"
}
```

### Project Structure

```bash
.
├── data
│   ├── certs
│   │   ├── intermediate-ca.crt
│   │   └── root-ca.crt
│   ├── config
│   │   └── ca.json
│   ├── db
│   └── secrets
│       ├── intermediate-ca.key
│       └── password
└── docker-compose.yml
```

Set permissions for directories and files:
```bash
chown -R 1000:1000 data
echo "<somegeneratedpass>" > data/secrets/password
```

Run step-ca:
```bash
docker compose up -d
```
Health check:
```bash
curl https://localhost:9000/health
# Expected output: {"status":"ok"}
```
Provisioners check:
```bash
curl https://acme-ca.home.arpa:9000/provisioners | jq
```

Logs if something fails:
```bash
docker compose logs -f
```

## Traefik Configuration

Example `docker-compose.yml` for Traefik with root CA trust:  
Main ting it use enviroment **LEGO_CA_CERTIFICATES**   
```yml
services:
  traefik:
    image: traefik
    container_name: traefik
    restart: always
    security_opt:
      - no-new-privileges:true
    ports:
      - 80:80
      - 443:443
    command:
      - "--providers.docker.network=webproxy"
    environment:
      - LEGO_CA_CERTIFICATES=/etc/ssl/certs/root-ca.crt
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/traefik.yml:ro
      - ./data/acme.json:/opt/traefik/acme.json
      - ./data/custom/:/custom/:ro
      - ./data/ssl/root-ca.crt:/etc/ssl/certs/root-ca.crt:ro
      - ./data/basic.auth:/basic.auth
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=https"
      - "traefik.http.routers.traefik.rule=Host(`traefik.${DOMAIN}`)"
      - "traefik.http.routers.traefik.tls=true"
      - "traefik.http.routers.traefik.tls.certresolver=stepca"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.services.traefik-traefik.loadbalancer.server.port=443"
      - "traefik.http.routers.traefik.middlewares=traefik-auth"
      - "traefik.http.middlewares.traefik-auth.basicAuth.usersFile=/basic.auth"
      - "traefik.http.routers.http-catchall.rule=hostregexp(`{host:.+}`)"
      - "traefik.http.routers.http-catchall.entrypoints=http"
      - "traefik.http.routers.http-catchall.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
    networks:
      - webproxy

networks:
  webproxy:
    name: webproxy
    external: true
```
In traefik.yml add part about certresolver

```bash 
certificatesResolvers:
  stepca:
    acme:
      caServer: "https://acme-ca.home.arpa:9000/acme/acme/directory"
      email: "admin@home.arpa"
      storage: "/opt/traefik/acme.json"
      tlsChallenge: true

```
Set `acme.json` permissions:
```bash
chmod 600 acme.json
```

For more on `home.arpa`, see [RFC 8375](https://www.rfc-editor.org/rfc/rfc8375.html).

