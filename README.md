# Secrets Management
In the course [Network Security in Practice](https://hpi.de/studium/lehrveranstaltungen/it-systems-engineering-ma/lehrveranstaltung/course/0/wintersemester-20182019-network-security-in-practice.html) at Hasso-Plattner Institute, we chose the topic Secrets Management to tackle the question of how secrets can be distributed, updated, and revoked in a highly distributed world. The technical report can be found [here](FIX_THIS_LINK).

# Pre-installation requirements
In order to follow the practical examples given in the report, you should `clone` this repository and have `docker` properly installed.

# Setup
To start the test-setup, please execute: `docker-compose up`. This will start a vault server, ssh server, sql db, and a jenkins server. For more details, checkout the [docker-compose.yml](docker-compose.yml).

## How to connect - SSH
Using the password `nsip`, you are able to connect to the running instance using: `ssh -p 3000 root@localhost`. If you are connected, you can use the `vault` command to access the running Vault server. You are also able to access the `mysql-client` from this instance.

## How to connect - Jenkins
To access the local jenkins server, use `http://localhost:8080/` and execute [jenkins_password.sh](jenkins_password.sh) to get the inial password.

# Practicals

In the following, we give an overview about the commands needed for some experience with Vault.

## Initializing and unsealing a Vault server

If we start Vault for the first time and want to see its status, we execute:

```bash
$ vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            n/a
HA Enabled         false
```

We see that we've connected to the Vault server, but it's not yet initialized. The next step is to initialize the Vault. We do that with: 

```bash
$ vault operator init
Unseal Key 1: SjIAQ/Mn2Y3iAVkpV6yQMdzSi31pvYY4GpjPrG9hVeh9
Unseal Key 2: KU69Q79D+yPYcZbgjBoWrZ3DzOSVxoOFexFfxZ128ymd
Unseal Key 3: 82FM3/lJfu08nkNCRAU46OKiSoG/S2v1C+zvObhPu7lv
Unseal Key 4: RR8/P5Pdf62GOo0hUnmeXwzLtKrnoeeHMfDnWMBugcSl
Unseal Key 5: xUuK56zppdyynP/aYAb6bdJ7uoWzV+74q3yQK5pA+0mG

Initial Root Token: s.GqmrzhH8vIfloC5SogFpli4a

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

This is a critical step that displays the unseal keys. Each unseal key is a shard of the master key. It's very important that these unseal keys be stored in a secure separate locations from each other. Now let's try to log in with the route token:

```bash
$ vault login s.GqmrzhH8vIfloC5SogFpli4a
Error authenticating: error looking up token

URL: GET http://vaultserver:8200/v1/auth/token/lookup-self
Code: 503. Errors:

* error performing token check: Vault is sealed
```

We get an error, because the Vault is currently sealed. Our next step is to unseal the Vault. In order to do that we use the command: 

```bash
$ vault operator unseal
Unseal Key (will be hidden):
```

Vault is needing three of the unseal keys to be unsealed. After we entered three keys, we see that sealed is false and our Vault is unsealed:

```bash
$ vault operator unseal
Unseal Key (will be hidden):
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.1.0
Cluster Name    vault-cluster-9c9491b2
Cluster ID      cce2ce83-8d3e-a9c9-a342-27598fd666df
HA Enabled      false
```

Now we are able to login using the route token.

## Dynamic Credentials


