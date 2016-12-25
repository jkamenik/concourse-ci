# concourse-ci

A basic setup for Concourse.  It uses [Ansible](https://www.ansible.com/) to install the basic components on the correct servers.  You have to provide the servers.  We assume the following:

1. All servers are connectable via SSH (either directly or via a jump host)
2. All servers are Ubuntu 16.04
3. You don't mind if packages are installed or removed from this server, or that the server is rebooted in the process of an update
4. Only the Web server is publicly accessible
5. The correct inbound ports are open for each server
  1. database: 5432 (private)
  2. web: 80 and 2222
  3. worker: N/A
6. The servers have no outbound restrictions
7. You have generated a set of keys to use
8. The ansible_user needs sudo access (with a password if fine)

If these are reasonable, then Ansible will pull the newest version of docker, pull the concourse docker container, and start them on each host.

# Basic layout

Concourse is made of 4 components: ATC, TSA, DB, and Worker.  The ATC and TSA need to be public so together are referred to as the "Web" server.  The ATC is a web server and job scheduler.  The TSA is an SSH server used to allow the Workers to securely register.  The DB is a standard Postgres install.  And the Workers is a process that will perform actual work on the machine.

Of the 4 only the Worker needs full and unsecured access to the machine.  This makes it possible for the worker to trash the machine it is running on, so generally it is best to run the working on a standalone VM which can be recreated if need be.

After concourse is running the `fly` cli command can be downloaded from the UI.  Once you have `fly` downloaded you will need it in your path.  All the pipelines are in the `./pipelines` directory.

# Providing the servers locally

Run `vagrant up` and `ansible-playbook generate-vagrant-inventory.yml` to get an inventory that will work with vagrant.  Every time you star the VM up you will need to regenerate the vagrant inventory.

To install concourse simply run `ansible-playbook -i /inventory/vagrant configure.yml`.  The vagrant VM will be running concourse.

# Providing the Servers for AWS

You should manually deploy the servers as follows:

1. Create a VPC with a private, but routable subnet
2. Create one EC2 instance with a public IP
  1. Make ports 22, 80, and 2222 externally accessible
3. Create one or more EC2 instances without pubic IPs.
  1. Make ports 22 accessible to the subject IPs.
4. Generate keys
5. Manually create the inventory (see below)
6. Copy SSL keys (optional)

## Generate keys

```bash
cd keys

ssh-keygen -t rsa -f ./tsa_host_key -N ''
ssh-keygen -t rsa -f ./session_signing_key -N ''
ssh-keygen -t rsa -f ./worker_key -N ''
```

## Creating the inventory

Add the public EC2 instance to the following groups:
1. web

Add each private EC2 instance to the following groups:
1. worker

## SSL Keys (Optional)

If you are have access to to a domain then you can add SSL keys to the `./keys` directory and they will be installed on the web node.  The keys need to be named as follows:

1. cert.crt - The webserver certificate in PEM format
2. intermediate.crt - The signer's certificate in PEM format
3. private.key - The private key
