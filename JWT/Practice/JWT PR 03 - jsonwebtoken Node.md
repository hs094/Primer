# PR 03 — jsonwebtoken (Node)

🔑 **One-line: Auth0's `jsonwebtoken` is the de facto Node.js JWT library — `sign`, `verify`, `decode`, with sync and async variants.**

The Node-side counterpart to [[JWT PR 01 - PyJWT]]. Used by Express/Fastify backends, Lambda authorizers, and most Node identity stacks. See also [[JWT PR 02 - joserfc Python]], [[JWT PR 04 - jwt.io Debugger]].

## Install

```bash
npm install jsonwebtoken
```

```bash
# Types for TypeScript
npm install --save-dev @types/jsonwebtoken
```

## Sign

Synchronous (no callback):

```javascript
const jwt = require('jsonwebtoken');

const token = jwt.sign({ foo: 'bar' }, 'shhhhh');
```

With explicit options ([[JWT FN 03 - Registered Claims]]):

```javascript
const token = jwt.sign(
  { sub: 'user-123', role: 'admin' },
  'shhhhh',
  {
    algorithm: 'HS256',
    expiresIn: '1h',           // or seconds: 3600
    notBefore: '0s',
    audience: 'api.example.com',
    issuer: 'auth.example.com',
    subject: 'user-123',
    jwtid: 'unique-id',
  }
);
```

RSA ([[JWT AL 02 - RSA Signatures]]):

```javascript
const fs = require('fs');
const privateKey = fs.readFileSync('private.key');

const token = jwt.sign(
  { foo: 'bar' },
  privateKey,
  { algorithm: 'RS256' }
);
```

Async (callback) form:

```javascript
jwt.sign({ foo: 'bar' }, privateKey, { algorithm: 'RS256' }, (err, token) => {
  if (err) return console.error(err);
  console.log(token);
});
```

## Verify

```javascript
jwt.verify(token, 'shhhhh', (err, decoded) => {
  if (err) return console.error(err);
  console.log(decoded.foo);   // 'bar'
});
```

Synchronous form (throws on failure):

```javascript
try {
  const decoded = jwt.verify(token, publicKey, {
    algorithms: ['RS256'],
    audience: 'api.example.com',
    issuer: 'auth.example.com',
    clockTolerance: 5,         // seconds of skew allowed
    maxAge: '1h',
  });
} catch (err) {
  // see error types below
}
```

`complete: true` returns header + payload + signature:

```javascript
const { header, payload, signature } = jwt.verify(token, key, {
  algorithms: ['RS256'],
  complete: true,
});
```

## Decode (no verification)

⚠️ Inspection only — never use to gate authorization. Equivalent to PyJWT's `get_unverified_header`. See [[JWT FN 05 - Validating vs Verifying]].

```javascript
const decoded = jwt.decode(token);
const full = jwt.decode(token, { complete: true });
// { header: {...}, payload: {...}, signature: '...' }
```

## Supported algorithms

| Family | Algorithms | Spec |
|---|---|---|
| HMAC | `HS256`, `HS384`, `HS512` | [[JWT AL 01 - HMAC]] |
| RSA | `RS256`, `RS384`, `RS512` | [[JWT AL 02 - RSA Signatures]] |
| RSA-PSS | `PS256`, `PS384`, `PS512` | [[JWT JO 06 - JWA]] |
| ECDSA | `ES256`, `ES384`, `ES512` | [[JWT AL 03 - ECDSA]] |

Note: classic `jsonwebtoken` historically lagged on [[JWT AL 04 - EdDSA]] — for Ed25519 reach for `jose` (panva/jose) instead.

## Error types

```javascript
const { TokenExpiredError, JsonWebTokenError, NotBeforeError } = require('jsonwebtoken');

try {
  jwt.verify(token, key, { algorithms: ['RS256'] });
} catch (err) {
  if (err instanceof TokenExpiredError) {
    // { name: 'TokenExpiredError', message: 'jwt expired', expiredAt: Date }
  } else if (err instanceof NotBeforeError) {
    // { name: 'NotBeforeError', message: 'jwt not active', date: Date }
  } else if (err instanceof JsonWebTokenError) {
    // { name: 'JsonWebTokenError', message: 'invalid signature' | 'jwt malformed' | ... }
  }
}
```

## Common patterns

### Express middleware

```javascript
function requireAuth(req, res, next) {
  const auth = req.header('Authorization') || '';
  const token = auth.replace(/^Bearer /, '');

  try {
    req.user = jwt.verify(token, publicKey, {
      algorithms: ['RS256'],
      audience: 'api.example.com',
      issuer: 'auth.example.com',
    });
    next();
  } catch (err) {
    res.status(401).json({ error: err.message });
  }
}
```

### JWKS verification (jwks-rsa)

`jsonwebtoken` doesn't ship a JWKS client — pair with `jwks-rsa` for [[JWT UC 03 - OIDC ID Tokens]]:

```javascript
const jwksClient = require('jwks-rsa');

const client = jwksClient({
  jwksUri: 'https://example.com/.well-known/jwks.json',
  cache: true,
  rateLimit: true,
});

function getKey(header, callback) {
  client.getSigningKey(header.kid, (err, key) => {
    callback(err, key && key.getPublicKey());
  });
}

jwt.verify(token, getKey, { algorithms: ['RS256'] }, (err, decoded) => {
  // ...
});
```

## Footguns

⚠️ **Always specify `algorithms: [...]` on `verify`.** Without it, `jsonwebtoken` will accept any algorithm in the header — classic [[JWT SE 02 - Algorithm Confusion]]. Mixing HMAC and RSA in the allowlist is also dangerous: an attacker can sign with HMAC using your public key as the secret. See [[JWT SE 03 - Key Management]].

⚠️ **`jwt.decode` does no verification.** It's `JSON.parse(base64url)`. Don't use it for auth — only for reading `kid` before lookup.

⚠️ **`expiresIn` requires no `exp` in payload.** If both are present, you get `Bad "options.expiresIn" option the payload already has an "exp" property`.

⚠️ **`expiresIn` strings use `vercel/ms` syntax.** `'1h'`, `'2 days'`, `'10m'` — not seconds. Bare numbers ARE seconds. `expiresIn: 60` ≠ `expiresIn: '60'` (the latter is rejected).

⚠️ **No EdDSA support in classic `jsonwebtoken`.** For [[JWT AL 04 - EdDSA]] / Ed25519 use `panva/jose`.

⚠️ **Async callback vs sync.** `jwt.sign(payload, key, options, callback)` runs async if a callback is provided, otherwise sync. Don't accidentally pass a callback that you ignore — your token comes back via the callback, not the return value.

💡 **Takeaway:** Pin `algorithms`, never trust `decode`, pair with `jwks-rsa` for OIDC, and reach for `panva/jose` if you need EdDSA or modern JWE.
