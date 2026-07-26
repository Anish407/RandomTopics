# Certificates, PKCS, PFX, AWS Secrets Manager, and ASP.NET Core Kestrel

This README explains how certificates, private keys, PKCS standards, PFX files, PEM files, AWS Certificate Manager, AWS Secrets Manager, AWS KMS, Application Load Balancers, ECS, and ASP.NET Core Kestrel work together.

The main production architecture discussed is:

```text
Client or CloudFront
        |
        | HTTPS
        v
Private Application Load Balancer
        |
        | HTTPS
        v
Private ECS Fargate task
        |
        v
ASP.NET Core Kestrel
```

The goal is to understand both the configuration and the internal mechanics.

---

# 1. The core certificate model

A TLS server normally needs two related cryptographic objects:

```text
Certificate
    Contains the public key and identity metadata

Private key
    Secret key corresponding to the public key
```

The certificate can be shared publicly.

The private key must remain secret.

A server such as Kestrel needs the private key because it must prove during the TLS handshake that it owns the public key contained in the certificate.

---

# 2. What is an X.509 certificate?

An X.509 certificate is a signed data structure containing information such as:

- Subject
- Issuer
- Public key
- Serial number
- Valid-from date
- Expiration date
- Subject Alternative Names
- Key usage
- Extended key usage
- Certificate authority signature

Example:

```text
Subject:
  CN=orders-api.prod.internal

Subject Alternative Name:
  DNS:orders-api.prod.internal

Public Key:
  RSA or ECDSA public key

Key Usage:
  Digital Signature

Extended Key Usage:
  TLS Web Server Authentication
```

The certificate normally does not contain the private key.

---

# 3. Public key and private key relationship

The public and private keys are mathematically related:

```text
Private key
    |
    | corresponding public key
    v
Public key
```

The public key can be distributed through the certificate.

The private key remains on the server.

During TLS, the private key is used to prove possession of the certificate identity. The private key is never transmitted to the ALB or client.

---

# 4. The private key does not encrypt the certificate

This is incorrect:

```text
Private key encrypts the certificate
```

The certificate is public information and does not require confidentiality.

The certificate private key is mainly used for:

- Digital signatures during TLS handshakes
- Proving possession of the certificate identity
- Certain encryption or decryption operations in older TLS designs
- Signing data where the certificate is used for digital signatures

The private key is not the password used to open a PFX file.

---

# 5. What does PKCS mean?

PKCS means:

```text
Public-Key Cryptography Standards
```

PKCS is a family of standards originally published by RSA Laboratories.

Important PKCS standards include:

| Standard | Purpose |
|---|---|
| PKCS #1 | RSA key and signature formats |
| PKCS #7 | Cryptographic message format, later related to CMS |
| PKCS #8 | Private-key storage format |
| PKCS #10 | Certificate Signing Request format |
| PKCS #11 | Interface for hardware security modules and cryptographic tokens |
| PKCS #12 | Container for certificates and private keys |

The most relevant standards for this architecture are:

```text
PKCS #8
    Private-key format

PKCS #10
    Certificate Signing Request

PKCS #12
    PFX/P12 container
```

---

# 6. What is a PFX file?

PFX originally meant:

```text
Personal Information Exchange
```

A `.pfx` file is normally a PKCS #12 container.

These extensions usually represent the same format:

```text
server.pfx
server.p12
```

A PFX can contain:

```text
PFX / PKCS #12
├── Server certificate
├── Private key
├── Intermediate certificates
└── Optional certificate chain
```

The key point is that a PFX is not just a certificate.

It is a portable container that can include the certificate and its corresponding private key.

---

# 7. What is a PEM file?

PEM originally referred to:

```text
Privacy-Enhanced Mail
```

Today, PEM commonly means a Base64-encoded text representation surrounded by header and footer markers.

A PEM file can contain many different object types.

Certificate:

```text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

Private key:

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

Encrypted private key:

```text
-----BEGIN ENCRYPTED PRIVATE KEY-----
...
-----END ENCRYPTED PRIVATE KEY-----
```

Certificate request:

```text
-----BEGIN CERTIFICATE REQUEST-----
...
-----END CERTIFICATE REQUEST-----
```

Therefore:

```text
PEM does not mean private key.
PEM does not mean certificate.
PEM is an encoding and packaging style.
```

The content between the markers determines what the PEM contains.

---

# 8. PFX versus PEM

## PFX

```text
One binary file
May contain certificate + private key + chain
Usually password protected
Very common in .NET and Windows environments
```

Example:

```text
server.pfx
```

## PEM

```text
One or more text files
Certificate and private key are often separate
Very common in Linux, Nginx, Envoy, and OpenSSL
```

Examples:

```text
server.crt
server.key
chain.pem
```

Comparison:

| Feature | PFX / PKCS #12 | PEM |
|---|---|---|
| Encoding | Binary | Base64 text |
| Certificate | Yes | Yes |
| Private key | Yes | Yes |
| Chain | Yes | Yes |
| Common packaging | One file | Multiple files |
| Password protection | Common | Optional |
| Common in .NET | Yes | Supported |
| Common in Linux tooling | Supported | Very common |

---

# 9. What is PKCS #8?

PKCS #8 is a format for private keys.

Example:

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

or:

```text
-----BEGIN ENCRYPTED PRIVATE KEY-----
...
-----END ENCRYPTED PRIVATE KEY-----
```

PKCS #8 is not a certificate container.

It contains private-key information.

By comparison:

```text
PKCS #8
    Private key

PKCS #12 / PFX
    Certificate + private key + optional chain
```

---

# 10. What is a CSR?

CSR means:

```text
Certificate Signing Request
```

A CSR is commonly represented using PKCS #10.

It contains:

- Public key
- Requested subject
- Requested DNS names
- Requested extensions
- A signature created using the private key

The private key is generated before the CSR.

Flow:

```text
Generate private key
        |
        v
Create CSR containing public key
        |
        v
Send CSR to CA
        |
        v
CA signs and returns certificate
```

The private key does not leave the system that generated it.

---

# 11. What does the PFX password do?

The PFX password does not generate the private key.

The PFX password does not derive the certificate private key.

Instead:

```text
PFX password
      |
      v
Password-based key derivation
      |
      v
Symmetric encryption key
      |
      v
Protect private-key material inside the PFX
```

Conceptually:

```text
PFX
├── Public certificate
├── Encrypted private-key material
└── Integrity information
```

The PFX password protects the sensitive contents of the PKCS #12 container, especially the private key.

It may also protect integrity information and other encrypted bags inside the container.

---

# 12. What does the certificate private key do?

The certificate private key is used by the TLS server.

For Kestrel, it is used to prove:

```text
I possess the private key corresponding to
the public key in this certificate.
```

The private key is not sent to the client.

The server performs private-key operations locally.

---

# 13. What happens during a TLS handshake?

For this architecture:

```text
ALB
  |
  | TLS connection
  v
Kestrel
```

A simplified TLS handshake is:

```text
1. ALB opens a TCP connection to the ECS task.

2. ALB sends ClientHello.
   This includes supported TLS versions and cipher suites.

3. Kestrel returns ServerHello.

4. Kestrel sends its public certificate.

5. Kestrel uses the private key internally to sign
   handshake data and prove possession of the key.

6. ALB and Kestrel derive shared symmetric session keys.

7. Future HTTP data is encrypted using symmetric encryption.

8. .NET decrypts the traffic before ASP.NET Core processes it.
```

The certificate private key is mainly used during authentication and key establishment.

The bulk application traffic is encrypted using temporary symmetric session keys because symmetric encryption is much faster.

---

# 14. Why does .NET need access to the private key?

Kestrel cannot serve HTTPS using only a public certificate.

It needs:

```text
Certificate + corresponding private key
```

In .NET, this is commonly represented using:

```csharp
X509Certificate2
```

You can check whether a loaded certificate has access to its private key:

```csharp
if (!certificate.HasPrivateKey)
{
    throw new InvalidOperationException(
        "The TLS certificate does not contain a private key.");
}
```

Without the private key, Kestrel cannot complete the server side of the TLS handshake.

---

# 15. Does .NET need the unencrypted private key?

Operationally, yes.

At runtime, .NET must have usable access to the private key.

That does not mean the key is stored unencrypted.

The normal flow is:

```text
Stored:
    Encrypted in Secrets Manager
    Protected inside password-protected PFX

Runtime:
    Secrets Manager returns plaintext secret data
    .NET opens PFX using password
    Private key becomes usable by cryptographic provider
```

The key may exist in process memory or inside an operating-system cryptographic provider.

For a normal PFX loaded into a Linux container, .NET must be able to use the decrypted private-key material.

---

# 16. AWS KMS and PFX password are different layers

There are three separate cryptographic elements:

```text
AWS KMS key
PFX password
Certificate private key
```

They have different purposes.

## AWS KMS key

Used by Secrets Manager to encrypt the stored secret at rest.

## PFX password

Used by .NET or OpenSSL to open the PKCS #12 container.

## Certificate private key

Used by Kestrel during TLS.

Layered view:

```text
Secrets Manager secret
    |
    └── encrypted at rest using AWS KMS
            |
            └── JSON containing Base64 PFX and password
                    |
                    └── PFX containing private key
```

---

# 17. Why use a PFX password if KMS already encrypts the secret?

KMS protects the Secrets Manager value while stored in AWS.

The PFX password protects the private key if the PFX file becomes separated from Secrets Manager.

Examples:

- Temporary PFX file accidentally copied
- CI workspace not cleaned
- File appears in a backup
- PFX transferred between environments
- PFX downloaded without the password
- Build artifact accidentally contains only the PFX

The PFX password travels with the file format as a protection mechanism.

However, when both the PFX and password are stored in the same Secrets Manager secret, the PFX password adds limited protection against a full Secrets Manager compromise.

The strongest controls remain:

- IAM
- Secrets Manager
- KMS
- No logging of secret values
- No certificate in container image
- Short certificate validity
- Automated rotation
- Restricted ECS task role

---

# 18. What should be stored in Secrets Manager?

A practical structure is:

```json
{
  "pfxBase64": "MIIK...",
  "password": "random-password"
}
```

This JSON is stored as the Secrets Manager `SecretString`.

Important:

```text
pfxBase64 is not only the public certificate.
```

It is the Base64 representation of the complete binary PFX file.

That PFX may contain:

- Certificate
- Private key
- Certificate chain

The JSON is only a wrapper.

---

# 19. What is Base64?

Base64 converts binary bytes into text.

```text
server.pfx binary bytes
        |
        | Base64 encoding
        v
MIIKCSqGSIb3DQEHAaCC...
```

Base64 is not encryption.

Anyone with the Base64 value can decode it back into the original PFX bytes.

The security comes from:

- Secrets Manager encryption
- KMS
- IAM
- PFX password
- TLS during retrieval

---

# 20. Secrets Manager representation

The structure is:

```text
Secrets Manager SecretString
└── JSON
    ├── pfxBase64
    │   └── Base64 representation of complete PFX
    └── password
        └── Password used to open the PFX
```

A more complete production secret could include metadata:

```json
{
  "pfxBase64": "MIIK...",
  "password": "random-password",
  "subject": "orders-api.prod.internal",
  "createdAtUtc": "2026-07-26T08:00:00Z",
  "expiresAtUtc": "2027-01-22T08:00:00Z",
  "serialNumber": "1234567890",
  "thumbprintSha256": "ABCDEF..."
}
```

Do not log the `pfxBase64` or `password`.

---

# 21. What happens when the application retrieves the secret?

The sequence is:

```text
1. ECS task starts.

2. Application receives temporary AWS credentials
   through the ECS task role.

3. Application calls Secrets Manager GetSecretValue.

4. Secrets Manager verifies IAM permissions.

5. Secrets Manager uses KMS internally.

6. Secrets Manager returns plaintext SecretString over TLS.

7. Application parses JSON.

8. Application Base64-decodes pfxBase64.

9. .NET opens the PFX using the password.

10. .NET creates an X509Certificate2 with private-key access.

11. Kestrel uses that certificate on the HTTPS endpoint.
```

The application does not normally handle the KMS key directly.

AWS performs the KMS operation inside the Secrets Manager service.

---

# 22. ECS task role versus task execution role

These roles are different.

## Task execution role

Used by ECS infrastructure for operations such as:

- Pulling images from ECR
- Sending logs
- Injecting task-definition secrets
- Performing ECS-agent-related startup operations

## Task role

Used by application code inside the container.

If the .NET application calls Secrets Manager directly, the permission belongs on the task role:

```text
secretsmanager:GetSecretValue
```

If a customer-managed KMS key is used, the task may also need:

```text
kms:Decrypt
```

depending on the KMS key policy and access model.

---

# 23. Loading the PFX in .NET

Example secret model:

```csharp
public sealed record TlsSecret(
    string PfxBase64,
    string Password);
```

Load it:

```csharp
using System.Security.Cryptography;
using System.Security.Cryptography.X509Certificates;

byte[] pfxBytes =
    Convert.FromBase64String(secret.PfxBase64);

try
{
    X509Certificate2 certificate =
        X509CertificateLoader.LoadPkcs12(
            pfxBytes,
            secret.Password,
            X509KeyStorageFlags.EphemeralKeySet);

    if (!certificate.HasPrivateKey)
    {
        certificate.Dispose();

        throw new InvalidOperationException(
            "The PFX does not contain a private key.");
    }
}
finally
{
    CryptographicOperations.ZeroMemory(pfxBytes);
}
```

`EphemeralKeySet` asks .NET not to persist the imported private key into a long-lived user or machine certificate store.

This is a good fit for ephemeral Linux containers.

---

# 24. How Kestrel uses the certificate

Programmatic example:

```csharp
using System.Security.Cryptography.X509Certificates;

var builder = WebApplication.CreateBuilder(args);

X509Certificate2 certificate =
    await LoadCertificateAsync();

builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(8443, listenOptions =>
    {
        listenOptions.UseHttps(certificate);
    });
});

var app = builder.Build();

app.Lifetime.ApplicationStopped.Register(certificate.Dispose);

app.MapGet("/health", () => Results.Ok(new
{
    status = "healthy"
}));

await app.RunAsync();
```

Kestrel listens on:

```text
https://0.0.0.0:8443
```

The ALB target group forwards traffic to port `8443`.

---

# 25. Internal .NET request flow

The application does not manually decrypt HTTP messages.

The flow is:

```text
Encrypted network packets
        |
        v
Operating-system TLS implementation
        |
        v
.NET TLS stack
        |
        v
Kestrel
        |
        v
ASP.NET Core middleware
        |
        v
Controllers / Minimal APIs / CoreWCF
```

By the time ASP.NET Core middleware receives the request, TLS has already decrypted the network traffic.

On Linux, .NET commonly uses platform cryptography backed by OpenSSL-related libraries.

On Windows, .NET integrates with Windows cryptographic APIs such as SChannel and certificate stores.

---

# 26. Kestrel configuration using a PFX file

Configuration example:

```json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://0.0.0.0:8443",
        "Certificate": {
          "Path": "/certs/server.pfx",
          "Password": "do-not-hardcode-this"
        }
      }
    }
  }
}
```

Environment-variable equivalent:

```text
ASPNETCORE_Kestrel__Endpoints__Https__Url=https://+:8443
ASPNETCORE_Kestrel__Endpoints__Https__Certificate__Path=/certs/server.pfx
ASPNETCORE_Kestrel__Endpoints__Https__Certificate__Password=<secret>
```

For ECS, loading the PFX directly from Secrets Manager in memory can avoid writing the private key to disk.

---

# 27. Loading PEM files in .NET

Modern .NET can also load separate PEM certificate and key files:

```csharp
X509Certificate2 certificate =
    X509Certificate2.CreateFromPemFile(
        "/certs/server.crt",
        "/certs/server.key");
```

Conceptually:

```text
server.crt
    Public certificate

server.key
    Private key
```

PFX is often more convenient in .NET because both are contained in one package.

---

# 28. How ACM certificates work on an ALB

When a normal ACM certificate is attached to an ALB:

```text
ACM
  |
  v
ALB HTTPS listener
```

AWS manages the certificate private key.

Your application does not receive a PFX.

The private key remains inside AWS-managed infrastructure.

This is the preferred model for:

```text
Client or CloudFront → ALB
```

Benefits:

- Private key does not leave ACM
- Automatic renewal
- No application-side secret distribution
- No PFX handling
- No Kestrel involvement for the listener certificate

---

# 29. How certificates work between ALB and ECS

An HTTPS target group creates a second TLS connection:

```text
Client
   |
   | HTTPS connection 1
   v
ALB
   |
   | HTTPS connection 2
   v
ECS task / Kestrel
```

The ALB listener certificate protects the first connection.

The Kestrel certificate protects the second connection.

The target group must use:

```text
Protocol: HTTPS
Port: 8443
Target type: IP
```

The ECS task must listen on HTTPS port `8443`.

---

# 30. ALB backend certificate validation

For HTTPS target groups, an Application Load Balancer establishes TLS to the target.

However, ALB does not perform normal browser-style validation of the backend certificate.

It does not strongly validate:

- Certificate issuer
- Hostname
- Expiration
- Trust chain
- Certificate revocation

This means a self-signed certificate can be used for ALB-to-ECS encryption.

Important consequence:

```text
Self-signed certificate
Private CA certificate
Public CA certificate
```

all provide backend TLS encryption, but the ALB does not use the CA trust chain to strongly authenticate the ECS service.

---

# 31. Public ACM certificate, Private CA, or self-signed?

## Public ACM certificate

Best for:

- ALB listeners
- Publicly trusted endpoints
- Browser or standard client trust
- External clients

Not normally necessary for ALB-to-ECS backend TLS.

## AWS Private CA

Best for:

- Central private PKI
- Organizational trust
- Workload identity
- Mutual TLS
- Service-to-service certificate validation
- ECS Service Connect TLS
- Central certificate issuance and governance

## Self-signed certificate

Best for:

```text
Private ALB → private ECS
```

when the requirement is only:

```text
Encrypt traffic in transit
```

and no component validates backend service identity.

---

# 32. What is centrally managed private PKI?

A centrally managed private PKI means an organization controls certificate issuance through a governed certificate authority hierarchy.

Example:

```text
Organization Root CA
        |
        ├── Production Issuing CA
        |       |
        |       ├── orders-api certificate
        |       ├── payments-api certificate
        |       └── inventory-api certificate
        |
        └── Development Issuing CA
                |
                ├── dev-orders certificate
                └── dev-payments certificate
```

Central management controls:

- Who can request certificates
- Which DNS names can be requested
- Certificate validity periods
- Certificate algorithms
- Certificate templates
- Audit logs
- Revocation
- Rotation
- Production versus development trust boundaries
- Which systems trust which CAs

A private PKI is useful only when a client actually validates the certificate chain and identity.

---

# 33. Self-signed certificate versus private PKI

## Self-signed

```text
Service creates certificate
Service signs its own certificate
No common trust root
```

Good for encryption when certificate trust is not validated.

## Private PKI

```text
Organization CA signs service certificate
Clients trust organization CA
Clients validate certificate identity
```

Good for:

- Workload identity
- Server authentication
- Mutual TLS
- Central governance
- Enterprise trust

---

# 34. Production-grade certificate generation

A production workflow should:

```text
1. Generate a strong random private key.

2. Generate a self-signed X.509 server certificate.

3. Add correct X.509 extensions.

4. Generate a strong random PFX password.

5. Package certificate and private key into PFX.

6. Validate the PFX.

7. Store Base64 PFX and password in Secrets Manager.

8. Force an ECS rolling deployment.

9. Verify ALB HTTPS health checks.

10. Retain the previous secret version for rollback.
```

---

# 35. Recommended X.509 extensions

A production server certificate should include:

```text
basicConstraints:
  CA:FALSE

keyUsage:
  digitalSignature
  keyEncipherment

extendedKeyUsage:
  serverAuth

subjectAltName:
  DNS:orders-api.prod.internal
```

`CA:FALSE` means the certificate is not allowed to act as a certificate authority.

`serverAuth` identifies it as a TLS server certificate.

The SAN identifies the expected DNS name.

Even though ALB does not validate the backend hostname, including a correct SAN is good certificate hygiene.

---

# 36. Generating a self-signed certificate and PFX with OpenSSL

Example:

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
umask 077

WORK_DIR="$(mktemp -d)"
trap 'rm -rf "${WORK_DIR}"' EXIT

DNS_NAME="orders-api.prod.internal"
PASSWORD_FILE="${WORK_DIR}/pfx-password.txt"
PRIVATE_KEY_FILE="${WORK_DIR}/server.key"
CERTIFICATE_FILE="${WORK_DIR}/server.crt"
PFX_FILE="${WORK_DIR}/server.pfx"
OPENSSL_CONFIG_FILE="${WORK_DIR}/openssl.cnf"

openssl rand -hex 32 > "${PASSWORD_FILE}"

cat > "${OPENSSL_CONFIG_FILE}" <<EOF
[req]
prompt = no
distinguished_name = distinguished_name
x509_extensions = server_extensions

[distinguished_name]
CN = ${DNS_NAME}
O = Example Organization
OU = Production Platform

[server_extensions]
basicConstraints = critical,CA:FALSE
keyUsage = critical,digitalSignature,keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @subject_alternative_names
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always

[subject_alternative_names]
DNS.1 = ${DNS_NAME}
EOF

openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:3072 \
  -out "${PRIVATE_KEY_FILE}"

openssl req \
  -new \
  -x509 \
  -sha256 \
  -days 180 \
  -key "${PRIVATE_KEY_FILE}" \
  -out "${CERTIFICATE_FILE}" \
  -config "${OPENSSL_CONFIG_FILE}"

openssl pkcs12 \
  -export \
  -out "${PFX_FILE}" \
  -inkey "${PRIVATE_KEY_FILE}" \
  -in "${CERTIFICATE_FILE}" \
  -name "orders-api" \
  -passout "file:${PASSWORD_FILE}"
```

---

# 37. Validate that the private key matches the certificate

```bash
PRIVATE_KEY_HASH="$(
  openssl pkey \
    -in server.key \
    -pubout \
    -outform DER |
  openssl sha256
)"

CERTIFICATE_KEY_HASH="$(
  openssl x509 \
    -in server.crt \
    -pubkey \
    -noout |
  openssl pkey \
    -pubin \
    -outform DER |
  openssl sha256
)"

if [[ "${PRIVATE_KEY_HASH}" != "${CERTIFICATE_KEY_HASH}" ]]; then
  echo "Certificate and private key do not match." >&2
  exit 1
fi
```

---

# 38. Validate the PFX

```bash
openssl pkcs12 \
  -in server.pfx \
  -passin file:pfx-password.txt \
  -noout \
  -info
```

Validate SAN:

```bash
openssl x509 \
  -in server.crt \
  -noout \
  -ext subjectAltName
```

Validate expiration:

```bash
openssl x509 \
  -in server.crt \
  -noout \
  -dates
```

Validate full certificate details:

```bash
openssl x509 \
  -in server.crt \
  -noout \
  -text
```

---

# 39. Convert PFX to Base64

Linux:

```bash
base64 -w 0 server.pfx > server-pfx-base64.txt
```

Portable alternative:

```bash
base64 < server.pfx | tr -d '\n'
```

Create JSON:

```json
{
  "pfxBase64": "MIIK...",
  "password": "random-password"
}
```

Do not print the real values into CI logs.

---

# 40. Store the JSON in Secrets Manager

Example:

```bash
aws secretsmanager put-secret-value \
  --secret-id "/prod/orders-api/tls" \
  --secret-string file://secret.json \
  --region eu-north-1
```

The secret should ideally be created by infrastructure as code.

The certificate-generation workflow should only update its value.

This allows tighter IAM permissions.

---

# 41. Recommended IAM separation

## Certificate rotation role

May have:

```text
secretsmanager:DescribeSecret
secretsmanager:PutSecretValue
ecs:UpdateService
```

It should be limited to the exact certificate secret and ECS service.

## ECS task role

May have:

```text
secretsmanager:GetSecretValue
kms:Decrypt
```

It should be limited to the exact TLS secret.

## Application task

Should not normally have:

```text
secretsmanager:PutSecretValue
secretsmanager:UpdateSecret
```

The application should consume the certificate, not rotate it.

---

# 42. Do not put the certificate in the Docker image

Never do this:

```dockerfile
COPY server.pfx /app/server.pfx
```

That places the private key into:

- Docker image layers
- ECR
- Build caches
- Developer workstations
- CI artifacts
- Image exports
- Security scanners

The certificate private key must be delivered at runtime.

---

# 43. Do not log sensitive values

Safe metadata to log:

- Subject
- Serial number
- Thumbprint
- Expiration date
- Secret version ID

Never log:

- Private key
- PFX Base64
- PFX password
- Complete Secrets Manager response
- Raw PFX bytes

---

# 44. Loading from Secrets Manager in .NET

Example:

```csharp
using System.Security.Cryptography;
using System.Security.Cryptography.X509Certificates;
using System.Text.Json;
using Amazon.SecretsManager;
using Amazon.SecretsManager.Model;

public sealed record TlsSecret(
    string PfxBase64,
    string Password);

public sealed class TlsCertificateProvider
{
    private readonly IAmazonSecretsManager _secretsManager;
    private readonly string _secretId;

    public TlsCertificateProvider(
        IAmazonSecretsManager secretsManager,
        string secretId)
    {
        _secretsManager = secretsManager;
        _secretId = secretId;
    }

    public async Task<X509Certificate2> LoadAsync(
        CancellationToken cancellationToken)
    {
        var response = await _secretsManager.GetSecretValueAsync(
            new GetSecretValueRequest
            {
                SecretId = _secretId,
                VersionStage = "AWSCURRENT"
            },
            cancellationToken);

        if (string.IsNullOrWhiteSpace(response.SecretString))
        {
            throw new InvalidOperationException(
                $"TLS secret '{_secretId}' has no SecretString.");
        }

        var secret = JsonSerializer.Deserialize<TlsSecret>(
            response.SecretString,
            new JsonSerializerOptions
            {
                PropertyNameCaseInsensitive = true
            });

        if (secret is null)
        {
            throw new InvalidOperationException(
                "TLS secret could not be deserialized.");
        }

        byte[] pfxBytes =
            Convert.FromBase64String(secret.PfxBase64);

        try
        {
            var certificate =
                X509CertificateLoader.LoadPkcs12(
                    pfxBytes,
                    secret.Password,
                    X509KeyStorageFlags.EphemeralKeySet);

            if (!certificate.HasPrivateKey)
            {
                certificate.Dispose();

                throw new InvalidOperationException(
                    "The TLS certificate has no private key.");
            }

            var now = DateTime.UtcNow;

            if (certificate.NotBefore.ToUniversalTime() > now)
            {
                certificate.Dispose();

                throw new InvalidOperationException(
                    "The TLS certificate is not valid yet.");
            }

            if (certificate.NotAfter.ToUniversalTime() <= now)
            {
                certificate.Dispose();

                throw new InvalidOperationException(
                    "The TLS certificate has expired.");
            }

            return certificate;
        }
        finally
        {
            CryptographicOperations.ZeroMemory(pfxBytes);
        }
    }
}
```

---

# 45. Configure Kestrel with the loaded certificate

```csharp
using Amazon.SecretsManager;
using System.Security.Cryptography.X509Certificates;

var builder = WebApplication.CreateBuilder(args);

var secretId =
    builder.Configuration["TLS_SECRET_ID"]
    ?? throw new InvalidOperationException(
        "TLS_SECRET_ID is not configured.");

using var secretsManager =
    new AmazonSecretsManagerClient();

var provider =
    new TlsCertificateProvider(
        secretsManager,
        secretId);

X509Certificate2 certificate =
    await provider.LoadAsync(
        CancellationToken.None);

builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(8443, listenOptions =>
    {
        listenOptions.UseHttps(certificate);
    });
});

var app = builder.Build();

app.Lifetime.ApplicationStopped.Register(
    certificate.Dispose);

app.MapGet("/health", () => Results.Ok(new
{
    status = "healthy"
}));

await app.RunAsync();
```

Environment variable:

```text
TLS_SECRET_ID=/prod/orders-api/tls
```

---

# 46. Security-group configuration

ALB security group:

```text
Outbound:
TCP 8443 to ECS task security group
```

ECS task security group:

```text
Inbound:
TCP 8443 from ALB security group only
```

Avoid allowing the entire VPC CIDR unless required.

Security-group references provide stronger workload-level segmentation.

---

# 47. ALB target group configuration

Recommended:

```text
Target type: IP
Protocol: HTTPS
Port: 8443
Health check protocol: HTTPS
Health check path: /health
Health check port: traffic port
```

ECS Fargate with `awsvpc` networking uses target type `IP`.

---

# 48. Certificate rotation

Certificate rotation and application deployment are different lifecycles.

Normal application deployment:

```text
Code change
    |
    v
Deploy new task
    |
    v
Task loads current certificate
```

Certificate rotation:

```text
Certificate near expiry
    |
    v
Generate new certificate
    |
    v
Update Secrets Manager
    |
    v
Force ECS rolling deployment
```

Do not depend only on application deployments for certificate rotation.

An application may remain unchanged for months.

---

# 49. Recommended rotation model

Example:

```text
Certificate validity:
  180 days

Rotate:
  At 120 to 150 days

Alarm:
  When fewer than 30 days remain
```

Sequence:

```text
1. Generate certificate.

2. Validate certificate and private key.

3. Create PFX.

4. Validate PFX password.

5. Store new secret version.

6. Force ECS deployment.

7. New tasks load new certificate.

8. HTTPS health check passes.

9. Old tasks drain.

10. Retain AWSPREVIOUS temporarily.
```

---

# 50. Secrets Manager versions

Secrets Manager uses staging labels such as:

```text
AWSCURRENT
AWSPREVIOUS
AWSPENDING
```

Simple rotation:

```text
New value becomes AWSCURRENT
Old value becomes AWSPREVIOUS
```

More controlled rotation:

```text
Generate AWSPENDING
        |
        v
Test certificate
        |
        v
Promote to AWSCURRENT
        |
        v
Deploy complete ECS service
```

---

# 51. Why not generate a new certificate on every deployment?

It can work, but it couples unrelated lifecycles.

Problems:

- Five deployments may create five unnecessary certificates.
- No deployments may mean no rotation.
- Rollback may unintentionally roll back certificate state.
- Pipeline retries may create excessive secret versions.
- Emergency certificate rotation should not require a code release.

A better model is:

```text
Normal deployment:
  Reuse current valid certificate

Certificate rotation:
  Independent scheduled or emergency workflow
```

A hybrid pipeline can rotate only if the certificate is near expiration.

---

# 52. Private-key exposure considerations

If the running container is fully compromised, an attacker may be able to access:

- Secrets Manager response
- PFX password
- Loaded certificate
- Private key in memory

Secrets Manager protects:

- Data at rest
- Controlled retrieval
- IAM authorization
- Encryption in transit

It does not protect against a fully compromised process that is already authorized to use the secret.

Additional controls are therefore essential:

- Minimal container image
- No shell where possible
- Read-only filesystem
- Least-privilege task role
- Restricted security groups
- Short certificate lifetime
- Automated rotation
- No secret logging
- Runtime hardening
- Dependency vulnerability management

---

# 53. Chiseled and distroless containers

A chiseled or distroless image may not include:

- Shell
- Package manager
- OpenSSL CLI
- Debugging tools

Kestrel can still load a PFX through .NET APIs if the runtime contains the required cryptographic libraries.

Recommended model:

```text
Generate and convert certificate outside application container
        |
        v
Store final PFX in Secrets Manager
        |
        v
Load PFX directly through .NET
```

Do not add OpenSSL and a shell to the runtime image only to generate certificates.

---

# 54. Certificate store versus direct loading

On Windows, certificates may be installed into stores such as:

```text
LocalMachine\My
CurrentUser\My
Trusted Root Certification Authorities
```

Applications can locate them by subject or thumbprint.

In Linux ECS containers, direct loading is often better:

```csharp
X509CertificateLoader.LoadPkcs12(...)
```

This avoids permanently installing the certificate into a machine-level store.

---

# 55. Server certificate versus trust store

These are different concepts.

## Server certificate

```text
Identifies the server
Contains or references private key
Used by Kestrel
```

## Trust store

```text
Contains trusted CA certificates
Used to validate other certificates
Normally contains no private keys
```

A Kestrel PFX should not automatically be added to the trusted-root store.

---

# 56. Public CA, private CA, and self-signed trust

## Public CA certificate

Trusted by standard browsers and operating systems.

## Private CA certificate

Trusted only by systems configured to trust the private CA.

## Self-signed certificate

Trusted only when explicitly configured, or when the client does not validate it.

For ALB HTTPS target groups, the ALB can establish TLS with self-signed target certificates because it does not perform normal trust validation.

---

# 57. Encryption versus authentication

These are different requirements.

## Encryption only

```text
Traffic must be unreadable in transit.
```

A self-signed certificate can satisfy this for ALB-to-ECS.

## Authentication

```text
The client must verify the identity of the server.
```

This requires certificate validation.

Examples:

- Validate CA chain
- Validate DNS name
- Validate expiration
- Validate revocation
- Possibly use mutual TLS

An ALB HTTPS target group does not provide strong backend certificate authentication.

---

# 58. Mutual TLS

Mutual TLS means both sides present certificates.

```text
Client presents certificate
Server validates client

Server presents certificate
Client validates server
```

This provides certificate-based identity in both directions.

Potential use cases:

- Service-to-service authentication
- Zero-trust internal networking
- Device identity
- Partner integration
- Regulated workloads

For ECS service-to-service communication, AWS Private CA and Service Connect TLS may be more appropriate than manually managed self-signed certificates.

---

# 59. Recommended production architecture

For the architecture:

```text
CloudFront
    |
    | HTTPS
    v
Private ALB
    |
    | HTTPS
    v
Private ECS Fargate
```

Recommended certificate choices:

```text
CloudFront or client → ALB:
    ACM public certificate attached directly to ALB

ALB → ECS:
    Self-signed service certificate in Secrets Manager
    when the goal is backend encryption only

ECS → ECS with verified service identity:
    AWS Private CA or another managed private PKI
```

---

# 60. Production checklist

## Certificate generation

- [ ] Strong RSA or ECDSA private key
- [ ] Correct SAN
- [ ] `CA:FALSE`
- [ ] `serverAuth`
- [ ] Strong random PFX password
- [ ] Certificate and key match
- [ ] PFX opens successfully
- [ ] Expiration is monitored

## Storage

- [ ] PFX not committed to Git
- [ ] PFX not embedded in Docker image
- [ ] PFX not uploaded as CI artifact
- [ ] Secret stored in Secrets Manager
- [ ] KMS encryption enabled
- [ ] Secret access is least privilege
- [ ] Secret values are never logged

## ECS

- [ ] Task role has only required read permission
- [ ] Task execution role is separated
- [ ] Tasks use private subnets
- [ ] Public IP disabled
- [ ] Security group allows HTTPS only from ALB security group
- [ ] Kestrel listens on dedicated HTTPS port
- [ ] HTTPS health check configured

## .NET

- [ ] Use `X509CertificateLoader`
- [ ] Verify `HasPrivateKey`
- [ ] Validate NotBefore and NotAfter
- [ ] Use `EphemeralKeySet` where supported
- [ ] Dispose certificate cleanly
- [ ] Zero raw PFX byte arrays where practical
- [ ] Do not log Secrets Manager responses

## Rotation

- [ ] Rotation independent of normal deployments
- [ ] Previous secret version retained temporarily
- [ ] ECS rolling deployment triggered
- [ ] ALB target health monitored
- [ ] Expiration alarm configured
- [ ] Emergency rotation procedure documented

---

# 61. Final mental model

```text
X.509 certificate
    Public identity document containing public key

Certificate private key
    Secret proof of ownership used during TLS

PEM
    Text encoding that can represent certificates or keys

PKCS #8
    Private-key format

PKCS #10
    Certificate Signing Request format

PKCS #12 / PFX
    Portable container holding certificate and private key

PFX password
    Protects private-key material inside the PFX

AWS KMS
    Protects Secrets Manager data at rest

AWS Secrets Manager
    Stores and controls access to the PFX and password

ECS task role
    Authorizes the application to retrieve the secret

X509Certificate2
    .NET representation of the loaded certificate

Kestrel
    Uses the certificate private key to terminate HTTPS

Application Load Balancer
    Opens an HTTPS connection to the ECS task
```

End-to-end flow:

```text
Certificate-generation workflow
        |
        | creates private key + certificate
        v
Password-protected PFX
        |
        | Base64 encoded
        v
Secrets Manager JSON
        |
        | retrieved using ECS task role
        v
.NET application
        |
        | decodes Base64
        | opens PFX using password
        v
X509Certificate2
        |
        v
Kestrel HTTPS endpoint
        |
        v
ALB sends encrypted traffic to ECS
```

---

# 62. Should a .NET application in a Linux ECS container use PEM or PFX?

Running the application in a Linux container does not force the application to use PEM.

Both formats work with modern .NET:

```text
Linux container
├── PFX / PKCS #12 supported by .NET
└── PEM certificate and private key supported by .NET
```

The more important question is:

```text
Which component terminates TLS?
```

For this architecture:

```text
Private ALB
    |
    | HTTPS
    v
Linux ECS Fargate container
    |
    v
ASP.NET Core Kestrel
```

Kestrel terminates TLS, so the .NET certificate APIs matter more than the traditional Linux convention.

## Recommended choice for .NET/Kestrel

Use a password-protected PFX when:

- The workload is primarily a .NET application.
- Kestrel terminates HTTPS.
- The certificate and private key should be rotated together.
- The application loads the certificate directly from Secrets Manager.
- No Nginx, Envoy, or other proxy requires separate PEM files.

A PFX bundles:

```text
PFX / PKCS #12
├── Server certificate
├── Corresponding private key
└── Optional certificate chain
```

This reduces the risk of accidentally combining:

```text
Certificate from version A
+
Private key from version B
```

The recommended Secrets Manager value is:

```json
{
  "pfxBase64": "MIIK...",
  "password": "random-password"
}
```

At runtime, .NET loads the PFX directly from memory:

```csharp
using System.Security.Cryptography;
using System.Security.Cryptography.X509Certificates;

byte[] pfxBytes =
    Convert.FromBase64String(secret.PfxBase64);

try
{
    X509Certificate2 certificate =
        X509CertificateLoader.LoadPkcs12(
            pfxBytes,
            secret.Password,
            X509KeyStorageFlags.EphemeralKeySet);

    if (!certificate.HasPrivateKey)
    {
        certificate.Dispose();

        throw new InvalidOperationException(
            "The PFX does not contain a private key.");
    }
}
finally
{
    CryptographicOperations.ZeroMemory(pfxBytes);
}
```

The application does not need:

- A Windows certificate store
- A permanently installed certificate
- A physical `.pfx` file written to disk
- OpenSSL inside the runtime container

The PFX can remain in memory after being retrieved from Secrets Manager.

## Why PEM is commonly associated with Linux

Linux-native infrastructure components often use separate PEM files.

Examples include:

```text
Nginx
Apache HTTP Server
Envoy
HAProxy
OpenSSL CLI
```

A common PEM layout is:

```text
server.crt
server.key
chain.pem
```

This is a software convention, not a Linux operating-system requirement.

A .NET application running on Linux can use PFX without any problem.

## When PEM is the better choice

Use PEM when:

- Nginx terminates TLS before forwarding to Kestrel.
- Envoy or another sidecar terminates TLS.
- The certificate provider already supplies PEM files.
- Your organization has standardized Linux workloads on PEM.
- The same certificate material must be consumed by non-.NET services.
- A service mesh or proxy requires separate certificate and key files.
- You deliberately want the certificate and private key as separate objects.

A Secrets Manager value could then look like:

```json
{
  "certificatePem": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
  "privateKeyPem": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----"
}
```

.NET can load them using:

```csharp
using System.Security.Cryptography.X509Certificates;

X509Certificate2 certificate =
    X509Certificate2.CreateFromPem(
        secret.CertificatePem,
        secret.PrivateKeyPem);
```

For an encrypted PEM private key, use the appropriate encrypted-PEM loading API and provide its password.

## PFX versus PEM for a Linux Kestrel container

| Area | PFX / PKCS #12 | PEM |
|---|---|---|
| Works in Linux container | Yes | Yes |
| Works with Kestrel | Yes | Yes |
| Certificate and key bundled | Yes | Usually separate |
| Easy atomic rotation | Better | Slightly harder |
| Easy to load directly in .NET | Excellent | Good |
| Needs Base64 in JSON SecretString | Yes | No |
| Password protection | Standard | Optional |
| Risk of key/certificate mismatch | Lower | Higher |
| Common for .NET-only services | Yes | Supported |
| Common for Nginx and Envoy | Less common | Yes |

## Recommended flow for this architecture

```text
Generate private key
        |
        v
Generate self-signed X.509 certificate
        |
        v
Package certificate + private key into PFX
        |
        v
Protect PFX with random password
        |
        v
Base64-encode complete PFX
        |
        v
Store PFX + password in Secrets Manager
        |
        v
ECS task retrieves secret using task role
        |
        v
.NET Base64-decodes PFX
        |
        v
X509CertificateLoader loads it with EphemeralKeySet
        |
        v
Kestrel serves HTTPS on port 8443
```

## Final decision

For a .NET 10 Kestrel application running in a Linux ECS container:

```text
Default:
    Use PFX

Use PEM:
    Only when another component, integration, or organizational standard
    provides a concrete reason
```

Linux alone is not a reason to prefer PEM over PFX.
