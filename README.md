# Secrets Management
In the course [Network Security in Practice](https://hpi.de/studium/lehrveranstaltungen/it-systems-engineering-ma/lehrveranstaltung/course/0/wintersemester-20182019-network-security-in-practice.html) at Hasso-Plattner Institute, we chose the topic Secrets Management to tackle the question of how secrets can be distributed, updated, and revoked in a highly distributed world. The technical report can be found [here](FIX_THIS_LINK).

# Pre-installation requirements
In order to follow the practical examples given in the report, you should `clone` this repository and have `docker` properly installed.

# Setup
To start the test-setup, please execute: `docker-compose up`. This will start a vault server, ssh server, sql db, and a jenkins server. For more details, checkout the [docker-compose.yml](FIX_THIS_LINK).

## How to connect - Vault
Using the password `nsip`, you are able to connect to the running instance using: `ssh -p 3000 root@localhost`. If you are connected, you can use the `vault` command to access the running Vault server.

## How to connect - Jenkins
To access the local jenkins server, use `http://localhost:8080/` and execute `jenkins_password.sh` to get the inial password.
