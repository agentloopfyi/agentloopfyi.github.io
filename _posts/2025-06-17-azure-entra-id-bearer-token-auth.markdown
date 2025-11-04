---
layout: post
comments: false
title:  "Authentication using Azure Entra ID bearer tokens"
date:   2025-06-17 10:00:00
---

You have an application and you want Azure Entra ID to manage your application security - Both user and application authentication.

### For application-to-application authentication

☑️ Azure Entra ID needs to identify your app and act as the auth provider, and can produce tokens for external apps on your behalf and help you verify the token sent over by the external app during API calls.

☑️ Application-to-application authentication meaning, your application exposes APIs to other apps and when those other apps consume APIs in your application, they need to obtain and send an `access_token` as **bearer token** which you then need to verify.

How do you set this up?

**Setting up application registration and client secrets**

Log into the Azure portal and go to app registraion.

Setup your app registration! This app registration represents your application for everything security. 

It will show a tenant ID (your org) and a client ID (representing your app).

Once done...

Click `Overview` ➡️ Set a meaningful Application ID URI, in dev we have set it as `api://<some-meaningful-text>`.

Go to `Certificates and Secrets` ➡️ Create a new client secret. This client secret should be unique for each consuming applications.

That means, for each external application that wants to consume your API, you need to create a client secret.

**Now how does the consuming app obtain a bearer token?**

Gather following info from your Azure Entra ID app registration both available on the Overview page of the app registration. 

```bash
tenant_id = *Directory (tenant) ID - Your org ID* 
client_id = *Application (client) ID - Your application ID as Entra understands it*
client_secret = *as obtained during creation*
scope = {Application ID URI}/.default
```

This needs to be securely used by the external app.

To get a token the external app must invoke the following API (Use Postman for example):

URL `POST https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token` 

Send request body as `x-www-form-urlencoded`:

```bash
grant_type=client_credentials
client_id=<generated-client-id>
client_secret=<generated-client-secret>
scope=api://<some-meaningful-text>/.default
```

Same way, by invoking this API, the external consumer will get an `access_token` (JWT).

Using the `access_token` as `Bearer <obtained-access-token>` in `Authorization` header, the external consumer will make API calls to your application.

**How do you verify the token**

The token was generated from your app registration in Entra by the external app, and they sent the token in Authentication header as a bearer token when they invoked one of your APIs.

How do you verify? You need to obtain the public keys from Entra, using which you can verify the bearer token.

You need a middleware. For example in Node.js you can create a simple middleware like below:

```bash
# Dependencies
npm i jsonwebtoken axios
```

```javascript
const axios = require('axios').default;
const jwt = require('jsonwebtoken');

const audience = 'api://<some-meaningful-text>'; // You set it earlier in Entra, this is the Application ID URI only
const tenantID = '<tenant-id>'; // Your org ID in Entra - Available on the overview page in app reg
const appID = '<client-id>'; // ID using which Entra identifies your app - Available on the overview page in app reg - interchangeably called client/app ID

const isTokenExpired = (epoch) => epoch*1000 < Date.now();

exports.checkAuthToken = async (req, res, next) => {
    try {
        // Extract token from the header
        const accessToken = ((req.headers['Authorization'] || req.headers['authorization']))?.split('Bearer ')[1];
        if (!accessToken) {
            res.status(401).send('Unauthorized');
        }

        // Decode the token
        const decoded = jwt.decode(accessToken, {complete: true});

        const kid = decoded.header.kid || undefined; // token tells which key-ID was used to sign it
        const alg = decoded.header.alg || undefined; // token tells which algorithm was used to sign it

        // Obtain public keys - Public keys will be always same (you can cache it also) - Will retrieve multiple keys
        const getPublicKeysEndpoint = `https://login.microsoftonline.com/${tenantID}/discovery/keys?appid=${appID}`;
        const pubKeysResponse = await axios.get(getPublicKeysEndpoint);
        const keys = pubKeysResponse.data.keys || [];

        // Identify the particular key among the public keys using the key-ID that you identified in the bearer token
        const publicKeysMap = {};
        keys.forEach(k => publicKeysMap[`${k.kid}`] = k.x5c[0] || undefined);

        const signingKey = publicKeysMap[`${kid}`];

        // Form well-structured signing key, and use this to verify the bearer token
        const formedSigningKey = `-----BEGIN CERTIFICATE-----\n${signingKey}\n-----END CERTIFICATE-----`;

        // Verify the token using the structured/well-formed key and algorithm
        const verified = jwt.verify(accessToken, formedSigningKey, { algorithms: [alg] })
        const exp = verified.exp || undefined;
        const audVerified = verified.aud || undefined;

        // Check for expiry
        if (exp && audVerified && !isTokenExpired(exp) && audVerified === audience) {
            return next();
        } else {
            res.status(401).send('Either token has expired, or wrong audience');
        }
    } catch (error) {
        logger.error('Error in checAuthToken: ', error);
        res.status(500).send('Internal Server Error, ' + error.message);
    }
}
```

**Steps**:

1. Middleware extracts the **bearer token** (JWT) from header
2. Decodes the JWT using jsonwebtoken library
3. Identifies key ID `kid` and algorithm `alg` from JWT header
4. Gets all public keys for this authentication server (authentication server is Entra/app registration set up for current environment)
`GET https://login.microsoftonline.com/{tenant_id}/discovery/keys?appid={client_id}`
5. May have multiple `kid`s pointing to multiple `x5c` keys
6. Find which `x5c` was can verify this token by matching with the `kid` obtained in **step #3**
7. Verify using well formed signed key and algorithm
8. Verify token is not expired and audience match (Audience match is not mandatory, added just as additional check)

**Code and explanations above shows how to establish application to application integration using `access_tokens` from Azure Entra.**

**This method doesn't have any user credentials involved in this.**

### For user sign-in (SSO) based authentication

☑️ If you want to enstablish an auth flow that involves user credentials, then you need `id_tokens`. 
 
☑️ In order to get an `id_token` you need to develop a user auth flow like MSAL or OIDC that will help you get an `id_token`.

If you are using Azure Entra ID (earlier known as Azure Active Directory), then you can best use MSAL.js to authenticate users to your application and you will get the `id_token`, along with `acess_token` and `refresh_token`.

You can use the `access_token` on behalf of the user to call the Graph API, like `https://graph.microsoft.com/oidc/userinfo` to get user info back. You can also use the `/me` API.

However, when you are making calls to your own backend APIs on behalf of the user, then from your front-end app you need to add the `Authentication` header in the API call and pass the `id_token` as `Bearer <id token>`.

Then you need a similar middleware that will intercept the API calls and validate the `id_token` against the public keys.

Working code in Python, shown as below:

```bash
# Dependencies
uv add pyjwt[crypto] httpx async-lru
```

```python
import httpx
import jwt
import asyncio
from async_lru import alru_cache

from cryptography.x509 import load_pem_x509_certificate
from cryptography.hazmat.backends import default_backend

tenantID = 'your tenant ID'
clientID = 'your client ID'

X509_CERT_TEMPLATE = """
-----BEGIN CERTIFICATE-----
{x509_cert}
-----END CERTIFICATE-----
"""

@alru_cache
async def _get_entra_public_keys() -> list:
    entra_public_keys_endpoint = f'https://login.microsoftonline.com/{tenantID}/discovery/keys?appid={clientID}'

    async with httpx.AsyncClient() as client:
        response = await client.get(entra_public_keys_endpoint)
        if response.status_code == 200:
            _response = response.json()
            return _response['keys'] or []
        else:
            # End of the world exception :P
            raise Exception("Entra public keys not available")


async def validate_token(id_token) -> bool:
    public_keys: list = await _get_entra_public_keys() 
    header = jwt.get_unverified_header(id_token)
    kid = header.get('kid')
    alg = header.get('alg')

    x509_cert = None
    for key in public_keys:
        if key['kid'] == kid:
            x509_cert = key

    formed_cert = X509_CERT_TEMPLATE.format(x509_cert=x509_cert['x5c'][0]).strip()

    try:
        loaded_cert = load_pem_x509_certificate(formed_cert.encode(), default_backend())

        decoded_payload = jwt.decode(
            id_token,
            key=loaded_cert.public_key(),
            algorithms=[alg],
            audience=[clientID],
            options={
                "verify_signature": True, 
                "verify_exp": True,
                "verify_nbf": True,
                "verify_iat": True,
                "verify_aud": True,
                "verify_iss": True,
            }
        )
        # Successful decode means all options verified
        return True
    
    except Exception as e:
        print('Exception occured while decoding token: ', e)
        return False
    
```

Happy authenticating your APIs! 😎
