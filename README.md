# Pod Group IoT Safe2 PoC 

## General Concepts

The scope of this PoC is to facilitate TLS 1.3 PSK-Only mode connections between Pod SIMs and the Connio platform. Through this connection:

(1) SIM should be able to request device's (a.k.a asset) MQTT credentials to connect to the MQTT server (part of the platform);

(2) SIM should be able to create a new device on the platform;

(3) SIM should be able to write data on behalf of the device. [FUTURE]

## System Setup

The Connio platform is a multi-tenant system. In order to store different pre-shared keys per SIM per account, each key should be associated with an account, identity (i.e. ICCID), and a key through the following API:

```
POST https://api.connio.cloud/v3/sys/security/psk

{
    "identity": "049305930495403",
    "psk": [ 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79 ],
    "accountId": "{ACCOUNT_ID}"
}
```

Typically this request should be done by PodGroup's platform before a SIM being provisioned. In order to do this call, requester reuires a special key/secret pair from Connio.

Similarly, an existing association can be removed using the following request:

```
DELETE https://api.connio.cloud/v3/sys/security/psk

{
    "identity": "iccid"
}

```

