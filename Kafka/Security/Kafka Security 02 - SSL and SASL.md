# 02 — SSL and SASL

🔑 [[SSL]] = transport encryption (+ optional [[mTLS]] auth). [[SASL]] = pluggable authentication.

Sources:
- https://kafka.apache.org/42/security/encryption-and-authentication-using-ssl/
- https://kafka.apache.org/42/security/authentication-using-sasl/

## SSL — Keystores and Truststores

| File | Holds | On |
|---|---|---|
| `keystore.jks` | Broker's private key + cert | Broker (one per node) |
| `truststore.jks` | Trusted CA certs | Broker **and** client |

Generate broker keystore (PKCS12):

```bash
keytool -keystore server.keystore.jks -alias localhost \
  -validity 365 -genkey -keyalg RSA -storetype pkcs12

# import CA root into shared truststore
keytool -keystore truststore.jks -alias CARoot -import -file ca-cert
```

## SSL Broker Config

```properties
listeners=SSL://0.0.0.0:9093
ssl.keystore.location=/etc/kafka/ssl/server.keystore.jks
ssl.keystore.password=changeit
ssl.key.password=changeit
ssl.truststore.location=/etc/kafka/ssl/truststore.jks
ssl.truststore.password=changeit
security.inter.broker.protocol=SSL
```

## mTLS — `ssl.client.auth`

| Value | Behavior |
|---|---|
| `none` | Server cert only (TLS, no client auth) |
| `requested` | Asks for client cert; not enforced. ⚠️ False security — avoid. |
| `required` | Enforces [[mTLS]]; client principal = cert DN |

For `required` the broker truststore must contain the **client CA**.

## SSL Gotchas

- ⚠️ Hostname verification on by default (since 2.0). Disable only with `ssl.endpoint.identification.algorithm=` (empty) — and don't.
- ⚠️ Internal CAs often issue certs with **`serverAuth` only**. Brokers act as both server and client (inter-broker) → need `serverAuth, clientAuth`.
- ⚠️ Intermediate CAs: import the **full chain** into the truststore.
- 💡 Verify SANs: `openssl x509 -in cert.crt -text -noout` — wrong SAN = handshake fails on clients.

## SASL Mechanisms

| Mechanism | Use | Notes |
|---|---|---|
| `PLAIN` | username/password | ⚠️ Use **only** with `SASL_SSL` |
| `SCRAM-SHA-256`, `SCRAM-SHA-512` | Salted challenge-response, creds in ZK/KRaft store | Default for password-based prod use |
| `GSSAPI` | Kerberos | Enterprise / AD environments |
| `OAUTHBEARER` | OAuth 2 / OIDC | JWT-based, integrates with IdPs |
| Delegation Tokens | Lightweight shared secret | Issued by brokers, scoped per client |

## Mechanism Configs

```properties
# broker
sasl.enabled.mechanisms=SCRAM-SHA-512,OAUTHBEARER
listener.name.client.sasl.enabled.mechanisms=SCRAM-SHA-512

# client
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
```

## JAAS Inline (`sasl.jaas.config`)

```properties
# SCRAM (client)
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="alice" password="alice-secret";

# PLAIN (client)
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="alice" password="alice-secret";

# Kerberos (client)
sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
  useKeyTab=true storeKey=true keyTab="/etc/kafka/kafka.keytab" \
  principal="kafka/host@REALM";
```

💡 Inline `sasl.jaas.config` replaces the older external `kafka_server_jaas.conf` — prefer it.

## Tags
[[Kafka]] [[Security]] [[SSL]] [[SASL]] [[mTLS]]
