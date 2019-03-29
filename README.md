# Table of Contents
* [Secrets Management](#secrets-management)
* [Requirements](#pre-installation-requirements)
* [Setup](#setup)
  * [How to connect - SSH](#how-to-connect---ssh)
  * [How to connect - Jenkins](#how-to-connect---jenkins)
* [Practicals](#practicals)
  * [Initializing and unsealing a Vault server](#initializing-and-unsealing-a-vault-server)
  * [Dynamic credentials](#dynamic-credentials)
    * [Policies and credentials with the database secrets engine](#policies-and-credentials-with-the-database-secrets-engine)
  * [Secure introduction](#secure-introduction)
  * [Response wrapping](#response-wrapping)


# Secrets Management
In the course [Network Security in Practice](https://hpi.de/studium/lehrveranstaltungen/it-systems-engineering-ma/lehrveranstaltung/course/0/wintersemester-20182019-network-security-in-practice.html) at Hasso-Plattner Institute, we chose the topic Secrets Management to tackle the question of how secrets can be distributed, updated, and revoked in a highly distributed world. The technical report can be found [here](documentation/nsip_report_secrets_management.pdf).

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

## Dynamic credentials

In order to create dynamic credentials on demand, we need to setup the Database Secrets Engine first. We described the architecture to this scenario in our [report](documentation/nsip_report_secrets_management.pdf).

```bash
$ vault secrets enable database
Success! Enabled the database secrets engine at: database/
```

The next set of commands are rather lengthy, so we saved them in [setup\_commands.txt](mariadb/setup_commands.txt) under the `mariadb-folder`. The first command, writes a configuration to the database secrets engine. The path is `database/config/nsip-mariadb`, which is the name we have given to this configuration. The plug-in name is `mysql-database-plugin`, which is compatible with `mariadb`. The connection URL, is a templated connection string that is used by Vault to connect to the database. The username and password are injected when vault calls out to the database. `Mariadb:3306` is the URL. The `allowed_roles` are `datareader` and `datawriter`, which we will be using to connect to the database. These are the roles that are allowed to create credentials using this configuration, and the username and password, `root` and `mysql`, are the username and password to the mysql database used to make the connection. 

The next two commands create roles in the database secrets engine. The path here is `database/roles/datareader`, which is the name of the role. The db name `nsip-mariadb`, matches the configuration we created earlier. The creation statement, is this statement that is used to create the user in the database when credentials are generated, `CREATE USER '{{name}}', IDENTIFIED BY '{{password}}, GRANT SELECT}`. Grant select is a SQL command that grants read only access to a database. The `datawriter` is similar, except in this case it grants all, which is create, read, update, and delete. The next two parameters are the default time to live, and the max time to live. These tokens have a default time to live of one hour, which can be renewed up to 24 hours.

### Policies and credentials with the database secrets engine

Our next step is to upload `policies` for the [datareader](mariadb/datareader.hcl) and [datawriter]((mariadb/datawriter.hcl)). They're essentially the same, they both grant access to the path in the database secrets engine that generates the credentials. We can upload them and create credentials with the following commands:

```bash
$ vault policy write datareader datareader.hcl
Success! Uploaded policy: datareader

$ vault policy write datawriter datawriter.hcl
Success! Uploaded policy: datawriter

# Create a token for this role
$ vault token create -policy=datareader
Key                  Value
---                  -----
token                s.dZO7MF7oGlThDOfHLwP7wxuE
token_accessor       1Vv62YCBFnit20ciUMgRPSbO
token_duration       768h
token_renewable      true
token_policies       ["datareader" "default"]
identity_policies    []
policies             ["datareader" "default"]

# Login in as the datareader and generate credentials
$ vault read database/creds/datareader
Key                Value
---                -----
lease_id           database/creds/datareader/WWblpx3...
lease_duration     1h
lease_renewable    true
password           A1a-4zmkaOvK6PTD9izI
username           v-token-datareader-NrUg6RnIZX46o
```

Vault has generated a username and password that we can use to login to the database using the `MySQL CLI` to connect to the `MariaDB` database. If we try to create a database, the access is denied. The `datareader` does not have permissions to write data. But if we repeat these steps with the `datawriter`, we are able to manipulate data. 

## Secure introduction

Let's assume that we want to deploy an application with a static username and password. We might, for example, be connecting to a database that is not supported by the database secrets engine. How do we securely automate the process of delivering secrets to an application on deployment? We refer again to our [report](documentation/nsip_report_secrets_management.pdf) to get a more detailed overview of the applied architecture. But in general, we want to deploy credentials from Vault to a Jenkins server automatically.

The first thing we need to do is to install the Jenkins Vault plug-in by HashiCorp. The next steps involve Vault to enable Jenkins to authenticate: 

```bash
# Enable secrets engine
$ vault secrets enable -path=secret kv
Success! Enabled the kv secrets engine at: secret/

# Save demo credentials to Vault
$ vault kv put secret/nsip/demo username=dbUser
Success! Data written to: secret/nsip/demo

# Enable the approle authentication method 
$ vault auth enable approle
Success! Enabled approle auth method at: approle/
```

The next step is to upload a [policy](jenkins/jenkins_policy.hcl) that Jenkins can use to authenticate with approle and to read the secret. The two paths give the policy the ability to log in with approle and to read the path in the secrets engine where we saved our secret:

```bash
# Upload policy to Vault
$ vault policy write jenkins jenkins_policy.hcl
Success! Uploaded policy: jenkins

# Create a authentication role for Jenkins
$ vault write auth/approle/role/jenkins policies=jenkins
Success! Data written to: auth/approle/role/jenkins

# Get role ID for Jenkins analogous to a username
$ vault read auth/approle/role/jenkins/role-id
Key        Value
---        -----
role_id    9ffdab4c-2971-e0f5-039f-1490578cd283

# Get secret ID for Jenkins
$ vault write -f auth/approle/role/jenkins/secret-id
---                   -----
secret_id             fb0d010d-d613-758e-c8c3-8793f768e2ae
secret_id_accessor    39dfceb5-71b0-8193-3a62-986aff2af1ad
```

Now we can add these credentials to Jenkins and create a new freestyle project. We enable the Vault plugin within the build environment with the following configuration:

```
Vault Url: http://vaultserver:8200
Path: secret/nsip/demo
Environment variable: NSIP_SECRET
Keyname: username
```

To demonstrate that Jenkins is able to access the secret, we use a build step: 

```bash
echo $NSIP_SECRET > secret.txt
```

When we build the job, we have the `secret.txt`in our workspace. This was written by the script that we added and we can see our value for our secret written to this file. From here, we can use our imagination, these secrets can be injected into a container. Written to a configuration file, any variety of different mechanisms to inject the secret into an application.

## Response wrapping

If we use the `vault token create` command, it will return a token that we can use associated with a policy to authenticate to Vault: 

```bash
$ vault token create
Key                  Value
---                  -----
token                s.hH4FyBKpj1E9WHSPhjFsxiTc
token_accessor       fTomhUqKt1VBd8eFl0e5aSW2
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

Response wrapping uses the cubbyhole secrets engine and instead of returning the token that we used to authenticate, we get a temporary token called a wrapping token. The way to do that is to execute the same command with a different parameter:

```bash
$ vault token create -wrap-ttl=5m -policy=jenkins
Key                              Value
---                              -----
wrapping_token:                  s.CSI8ZXFkJr7HFQSiUTM9LQV1
wrapping_accessor:               RDVL7Pg5L7rsJwmKFnA47DdO
wrapping_token_ttl:              5m
wrapping_token_creation_time:    2019-03-29 10:07:31.5555671 +0000 UTC
wrapping_token_creation_path:    auth/token/create
wrapped_accessor:                WoWP9PMZchJcbN06PbfgiRaQ
```
This tells Vault to return a wrapping token and put the actual token into a cubbyhole for us to retrieve later. We can unwrap this by executing: 

```bash
$ vault unwrap s.CSI8ZXFkJr7HFQSiUTM9LQV1
Key                  Value
---                  -----
token                s.dzJcBRdnB3H6TvQR9Ilx9zd8
token_accessor       WoWP9PMZchJcbN06PbfgiRaQ
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

We get the actual token form the cubbyhole. Vault has now destroyed that cubbyhole. So the only location this token exists here is in our output. This gives an idea of how response wrapping can be used to add security for secure introduction. Jenkins or some other CI tool can request a wrapping token and inject that into an application build. The application can then use the wrapping token to unwrap the actual secret and store it in memory. Under those circumstances, the only place the actual token ever exists are in the cubbyhole and then in the application's memory.
