# ASS

## recon

This challenge gives you a web app that:
* recieve content from users on profile admin or client
* in admin profile if username is valid you will get certificate like this

  ```"certificate": "-----BEGIN CERTIFICATE-----\nMIIBDzCBwqADAgECAhRy7kH16jA8MUWfJC7R4gWXw5fFPDAFBgMrZXAwGTEXMBUG\nA1UEAwwOQVNTIElzc3VpbmcgQ0EwHhcNMjYwOTExMDQyNDMxWhcNMjYxMDExMDQy\nOTMxWjAQMQ4wDAYDVQQDDAVTU1NTUzAqMAUGAytlcAMhAH4kzlD/+f/X6J4eaJgZ\nW4L9ol95UqYk0g7IW2l3LQdEoyUwIzAMBgNVHRMBAf8EAjAAMBMGA1UdJQQMMAoG\nCCsGAQUFBwMCMAUGAytlcANBAImxKwYiZ76CsKQpTju3mXcS3uGbvG4oiUg1AHCs\no8nXkVhLU8n5a/kCypIHBKl9LDV4Vr4ur+PGHytGfpBdjAg=\n-----END CERTIFICATE-----\n"```

* in client profile if username is valid and not already in use you will get certificate and private key like this
  
  cert:
  
  ```"certificate": "-----BEGIN CERTIFICATE-----\nMIIBEDCBw6ADAgECAhRp4FV+ljLfptm8Z3Lteef7+J6J5zAFBgMrZXAwGTEXMBUG\nA1UEAwwOQVNTIElzc3VpbmcgQ0EwHhcNMjYwOTExMDQyNzM4WhcNMjYxMDExMDQz\nMjM4WjARMQ8wDQYDVQQDDAbhup5TU1MwKjAFBgMrZXADIQCaF/hLcYOmaY27BmLb\nAP0gXEUKO5xFeddmqqP0prB+paMlMCMwDAYDVR0TAQH/BAIwADATBgNVHSUEDDAK\nBggrBgEFBQcDAjAFBgMrZXADQQBJeLlXqTPCwTwJa7e/z6N3pO+d1MovNIGvv6Oi\nM3nlhgz+lNcGSyuTMo6FH1QGm7Map81lMwdKOnN3l3QIPDAC\n-----END CERTIFICATE-----\n"```
  
  private key :
  
  ```"private_key": "-----BEGIN PRIVATE KEY-----\nMC4CAQAwBQYDK2VwBCIEIMoierWmyFHADQzR7vepwObyPYqGojWhqyB5PNvzl5KW\n-----END PRIVATE KEY-----\n"```

Source code, app.py :

```python
import base64
import binascii
import os
import secrets
import threading
import time
from pathlib import Path
from typing import Literal

from cryptography import x509
from cryptography.exceptions import InvalidSignature
from cryptography.hazmat.primitives.asymmetric import ed25519
from fastapi import FastAPI, HTTPException
from fastapi.responses import HTMLResponse, PlainTextResponse
from pydantic import BaseModel

import ca
import names

FLAG = os.environ["FLAG"]

NONCE_LIFETIME = 120.0
MAX_NONCES = 512
MAX_CERTIFICATE_PEM = 8192

INDEX = (Path(__file__).parent / "templates" / "index.html").read_text(encoding="utf-8")

app = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)

authority = ca.CertificateAuthority()

lock = threading.Lock()
issued_names: set[str] = set()
administrator: x509.Certificate | None = None
nonces: dict[str, float] = {}


class CertificateRequest(BaseModel):
    profile: Literal["ADMIN", "CLIENT"]
    name: str


class CertificateResponse(BaseModel):
    profile: str
    name: str
    certificate: str
    private_key: str | None = None


class NonceResponse(BaseModel):
    nonce: str
    expires_in: int


class AdminRequest(BaseModel):
    certificate: str
    nonce: str
    signature: str


class AdminResponse(BaseModel):
    flag: str


@app.get("/", response_class=HTMLResponse)
def index() -> str:
    return INDEX


@app.get("/ca.pem", response_class=PlainTextResponse)
def ca_certificate() -> str:
    return ca.certificate_pem(authority.certificate)


@app.post(
    "/certificates",
    response_model=CertificateResponse,
    response_model_exclude_none=True,
)
def provision(request: CertificateRequest) -> CertificateResponse:
    global administrator

    if not names.is_acceptable(request.name):
        raise HTTPException(status_code=400, detail="invalid name")

    with lock:
        if request.name in issued_names:
            raise HTTPException(status_code=409, detail="name already in use")
        if request.profile == "ADMIN" and administrator is not None:
            raise HTTPException(
                status_code=409,
                detail="an administrator certificate has already been provisioned",
            )

        certificate, key = authority.issue(request.name)
        try:
            _ = ca.subject(certificate).hashable
        except ValueError:
            raise HTTPException(status_code=400, detail="invalid name") from None

        issued_names.add(request.name)
        if request.profile == "ADMIN":
            administrator = certificate

    return CertificateResponse(
        profile=request.profile,
        name=request.name,
        certificate=ca.certificate_pem(certificate),
        private_key=None if request.profile == "ADMIN" else ca.private_key_pem(key),
    )


@app.get("/auth/nonce", response_model=NonceResponse)
def issue_nonce() -> NonceResponse:
    nonce = secrets.token_hex(32)
    now = time.monotonic()
    with lock:
        for expired in [n for n, born in nonces.items() if now - born > NONCE_LIFETIME]:
            del nonces[expired]
        if len(nonces) >= MAX_NONCES:
            raise HTTPException(status_code=429, detail="too many outstanding nonces")
        nonces[nonce] = now
    return NonceResponse(nonce=nonce, expires_in=int(NONCE_LIFETIME))


def consume_nonce(nonce: str) -> bool:
    with lock:
        born = nonces.pop(nonce, None)
    return born is not None and time.monotonic() - born <= NONCE_LIFETIME


@app.post("/admin", response_model=AdminResponse)
def administration(request: AdminRequest) -> AdminResponse:
    with lock:
        administrator_certificate = administrator
    if administrator_certificate is None:
        raise HTTPException(
            status_code=409, detail="no administrator certificate has been provisioned"
        )

    if len(request.certificate) > MAX_CERTIFICATE_PEM:
        raise HTTPException(status_code=400, detail="certificate too large")
    try:
        presented = x509.load_pem_x509_certificate(request.certificate.encode())
    except ValueError:
        raise HTTPException(status_code=400, detail="malformed certificate") from None
    try:
        signature = base64.b64decode(request.signature, validate=True)
    except (binascii.Error, ValueError):
        raise HTTPException(status_code=400, detail="malformed signature") from None

    if not consume_nonce(request.nonce):
        raise HTTPException(status_code=401, detail="unknown or expired nonce")
    if not authority.issued_by_us(presented):
        raise HTTPException(
            status_code=401, detail="certificate was not issued by this authority"
        )
    if not ca.is_in_validity_period(presented):
        raise HTTPException(
            status_code=401, detail="certificate is not valid at this time"
        )

    public_key = presented.public_key()
    if not isinstance(public_key, ed25519.Ed25519PublicKey):
        raise HTTPException(status_code=401, detail="unsupported certificate key type")
    try:
        public_key.verify(signature, request.nonce.encode())
    except InvalidSignature:
        raise HTTPException(
            status_code=401, detail="signature does not verify"
        ) from None

    try:
        authorized = ca.subject(presented) == ca.subject(administrator_certificate)
    except ValueError:
        raise HTTPException(status_code=400, detail="malformed certificate") from None
    if not authorized:
        raise HTTPException(status_code=403, detail="not the administrator")

    return AdminResponse(flag=FLAG)

```

This part of code will return flag we will start from here :

```python
class AdminRequest(BaseModel):
    certificate: str
    nonce: str
    signature: str


class AdminResponse(BaseModel):
    flag: str
```

from this code we need to send 3 thing ```certificate```, ```nonce```, ```signature``` to 
```python
/admin
```
