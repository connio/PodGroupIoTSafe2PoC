# Pod Group IoT Safe2 PoC 

## General Concepts

The scope of this PoC is to facilitate TLS 1.3 PSK-Only mode connections between Pod SIMs and the Connio platform. Through this connection:

(1) SIM should be able to request device's (a.k.a asset) MQTT credentials to connect to the MQTT server (part of the platform);

(2) SIM should be able to create a new device on the platform;

(3) SIM should be able to write data on behalf of the device. [FUTURE]

## System Setup

TLS 1.3 NO-PSK endpoint for admin operations: `https://api.connio.cloud/v3/sys/security/psk`

The Connio platform is a multi-tenant system. In order to store different pre-shared keys per SIM per account, each key should be associated with an account, and an identity (i.e. ICCID) through the following API:

[1]
```
POST https://api.connio.cloud/v3/sys/security/psk

{
    "identity": "049305930495403",
    "psk": [ 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79 ],
    "accountId": "{ACCOUNT_ID}"
}
```

If successful, this request returns 1; 0 otherwise. `identity` should be unique globally. An `identity` can be overriden any time with a new PSK and/or account.

Typically this request should be done by Pod Group's platform before a SIM being distributed. In order to execute this command, a special key/secret pair should be issued by Connio to Pod Group. Single key/secret pair will be enough for all SIMs regardless of their account.

**Please note that the requests behind `/sys/..` path can be made from certain IPs only. For this PoC we removed this restriction. Key/Secret pair will be shared separately over Slack.**

Similarly, an existing association can be removed using the following request:

[2]
```
DELETE https://api.connio.cloud/v3/sys/security/psk

{
    "identity": "iccid"
}

```

If successful, this request returns 1; 0 otherwise.

## SIM Communication

TLS 1.3 PSK-ONLY endpoint for SIM operations: `https://api.connio.cloud:11111/v1/`

**Please note that the port used for PSK-Only mode endpoints is `11111`. **

Once the ICCID is associated with a PSK and account, the SIM is ready to communicate with the platform. Currently the following functionality is supported:

### Creating new device 

```
POST https://api.connio.cloud:11111/v1/dev/{DEVICE_ID}?iccid={ICCID}

```

This request doesn't carry any payload.



**By default, TLS13-AES128-CCM-SHA256 and TLS13-AES128-GCM-SHA256 cipher suites are supported.**

Following response should be expected if everything goes fine.

```
{
"cli": {CLIENT_ID},                 <--- automatically assigned by the platform 
"usr": {MQTT_CLIENT_USERNAME},      <--- MQTT credentials    
"pwd": {MQTT_CLIENT_PASSWORD}
}
```

In case of an error, SIM should get an error with HTTP 404 status code. We can improve error types in the future.


### Getting device MQTT credentials

```
GET /v1/dev/{DEVICE_ID}?iccid={ICCID}
```

This request doesn't carry any payload.

Following response should be expected if everything goes fine.

```
{
"cli": {CLIENT_ID},                 <--- automatically assigned by the platform 
"usr": {MQTT_CLIENT_USERNAME},      <--- MQTT credentials    
"pwd": {MQTT_CLIENT_PASSWORD}
}
```

## SIM Simulator Excerpts

```
server = 'api.connio.cloud'
port = 11111

ICCID = b'049305930495403'

def callback(data):
    nonlocal quit, sock
    print(data)
    if data == b"bye\n":
        quit = True

session = TLSClientSession(
    server_names=server, psk=psk, psk_label=ICCID, data_callback=callback, psk_only=True
)

.....

```

```
...


// Request device credentials where 48090435039402 is the device ID.
data = bytes('GET /v1/dev/48090435039402?iccid=049305930495403 HTTP/1.1\x0d\x0a' +
                 'Host: localhost:11111\x0d\x0a' +
                 'Content-Length: {0}\x0d\x0a\x0d\x0a'.format(0), 'utf-8')

...
```


```
...

// Create new device where 48090435039402 is the new device CID (customer ID).
 data = bytes('POST /v1/dev/48090435039402?iccid=049305930495403 HTTP/1.1\x0d\x0a' +
                 'Host: localhost:11111\x0d\x0a' +
                 'Content-Length: {0}\x0d\x0a\x0d\x0a'.format(0), 'utf-8')
                 
...
```


## Test Portal

A Pod Group account has been created for further testing on the [Connio demo cloud](https://app.connio.cloud).

You should have received an activation token by now to login. Please complete your registeration to conduct further tests.

ICCID `049305930495403` has already been created and associated with your account (\_acc_1254624011134164394)

Please let Baris or Emre know if you need quick walkthrough on the platform to easy navigatation.

## Misc

- Device credentials for MQTT server connection are generated during the device creation. Planned expiration strategies are `ON-DEMAND`, `AT-EVERY-REQUEST`, `AFTER-X-REQUESTS` and `TIME-BASED`. Only `ON-DEMAND` is implemented at the moment where the credentials can be manually (via UI) or programmatically re-generated if needs be.

- There is a 250 bytes limit for each message between the SIM and the platform. If the generated response is larger than 250 bytes, the platform doesn't send any response and records the incident as a WARNING in the log.

- This feature should be considered as Alpha. Request and Response validations are not complete.

- [IMPROVEMENT] `Create new device` request can be improved by adding a `config` parameter such as:

```
GET /v1/dev/{DEVICE_ID}?iccid={ICCID}&config={CONFIG_PROPERTY_NAME}
```

In such case, the response would be:

```
{
"cli": {CLIENT_ID}, 
"usr": {MQTT_CLIENT_USERNAME},
"pwd": {MQTT_CLIENT_PASSWORD},
"config": { ... }
}
```

- [IMPROVEMENT] The Connio platform support device templating called **Device Profile**. `Create new device` request can be improved by adding a `profile` parameter such as:

```
GET /v1/dev/{DEVICE_ID}?iccid={ICCID}&profile={PROFILE_NAME_OR_ID}
```

In such case, the response would be same as the current one but the device would be generated using the given template behind the sceen.

```
{
"cli": {CLIENT_ID}, 
"usr": {MQTT_CLIENT_USERNAME},
"pwd": {MQTT_CLIENT_PASSWORD}
}
```

