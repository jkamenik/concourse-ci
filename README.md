# concourse-ci

A basic setup for Concourse.  It uses [Ansible](https://www.ansible.com/) to install the basic components on the correct servers.  You have to provide the servers.  We assume the following:

1. All servers are connectable via SSH (either directly or via a jump host)
2. All servers are Ubuntu 16.04
3. You don't mind if packages are installed or removed from this server, or that the server is rebooted in the process of an update
4. Only the Web server is publicly accessible
5. The correct inbound ports are open for each server
  1. database: 5432 (private)
  2. web: 22, 80, 2222
  3. worker: 22
6. The servers have no outbound restrictions
7. You have generated a set of keys to use
8. The ansible_user needs sudo access (with a password if fine)

If these are reasonable, then Ansible will pull the newest version of docker, pull the concourse docker container, and start it on the web host.  The worker will pull the newest version of concourse and start it directly.

# Basic setup

Concourse is made of 4 components: ATC, TSA, DB, and Worker.  The ATC and TSA need to be public so together are referred to as the "Web" server.  The ATC is a web server and job scheduler.  The TSA is an SSH server used to allow the Workers to securely register.  The DB is a standard Postgres install.  And the Workers is a process that will perform actual work on the machine.

Of the 4 only the Worker needs full and unsecured access to the machine.  This makes it possible for the worker to trash the machine it is running on, so generally it is best to run the working on a standalone VM which can be recreated if need be.

After concourse is running the `fly` cli command can be downloaded from the UI.  Once you have `fly` downloaded you will need it in your path.  All the pipelines are in the `./pipelines` directory.

## Generate keys

Concourse needs a set of (at least) 3 public/private key pairs.  They are need to be in the keys directory, and when named correctly ansible will copy them to the correct hosts.

1. tsa_host_key:
  1. public:  Used the worker to ensure they are connecting to the correct TSA
  2. private: Used by the TSA as the server signature
2. session_signing_key:  Used the ATC for Oauth Authentication
3. worker_key:
  1. public:  Used by the TSA to ensure the worker can register (pre-shared)
  2. private: Used by the worker as the server signature

```bash
cd keys

ssh-keygen -t rsa -f ./tsa_host_key -N ''
ssh-keygen -t rsa -f ./session_signing_key -N ''
ssh-keygen -t rsa -f ./worker_key -N ''
```

## Create a team

The default team is "main" which has full admin rights to the server.  The username/password for the "main" team is set by ansible during install.  You should *NOT* give users access to main.  Instead you should create one or more new teams.

```
fly -t main login -c http://<concourse-ip>
fly -t main set-team -n <team-name> --no-really-i-dont-want-any-auth
fly -t <team-name> login -c http://<concourse-ip> -n <team-name>
```

## Create a pipeline

Some sample pipelines are in the pipelines directory.  Load them as follows:

```bash
$ fly -t <team-name> set-pipeline -p <pipeline-name> -c pipelines/<pipeline-to-load>.yml
```

## Debug internal problems

Most of the time if there is an issue with the build it is because of the repo and the error messages captured as part of the build are enough to debug the issue.  Sometimes the issue is actually with Concourse.

### Check that there are workers and containers

Most of the time the reason a build isn't doing anything is that there is are no workers or containers available to do the work.

```bash
$ fly -t <team-name> workers
name           containers  platform  tags  team
ubuntu-xenial  5           linux     none  none

$ fly -t fourv containers
handle                                ttl       validity  worker         pipeline   job                build #  build id  type   name               attempt
34c0a5ad-857e-47ac-45df-b41712e1117a  00:03:17  00:05:00  ubuntu-xenial  sample-pr  none               none     none      get    concourse-pr-test  n/a
4ad5c975-0743-4047-5e82-ea813e6431a2  00:03:15  00:05:00  ubuntu-xenial  sample-pr  none               none     none      check  concourse-pr-test  n/a
511a2fa8-23b6-49dc-5541-7f9f274d8c3c  00:04:36  00:05:00  ubuntu-xenial  sample-pr  none               none     none      check  concourse-pr-test  n/a
ab44d3ce-c4d1-4eb5-4ab1-04e5e54e9ba3  00:04:44  00:05:00  ubuntu-xenial  sample-pr  concourse-pr-test  1        1         get    concourse-pr-test  n/a
b101db8c-7b5f-47af-56fd-b8c8bb46e149  00:04:44  00:05:00  ubuntu-xenial  sample-pr  concourse-pr-test  1        1         check  concourse-pr-test  n/a
```

### Check the resource is running correctly

Assuming there are workers and containers for the given resource then checking the behavior of the resource will provide the details needed to fix the issue.  

The following will force a new check.

```bash
$ fly -t <team-name> check-resource -r <pipeline>/<resource>
... Any error output if the check fails ...
```

The following will attach to the resource container directly and allow you to execute commands.  Note: do this may invalidate the container and there is currectly no easy to remove broken containers.  Proceed with caution.

```bash
$ fly -t <team-name> hijack -c <pipeline>/<resource>
# echo '{"args": "..."}' | /opt/resource/check
... output of a check ...
```

# Providing the servers locally

Run `vagrant up web worker`, and `ansible-playbook generate-vagrant-inventory.yml` to get an inventory that will work with vagrant.  Every time you start the VMs up you will need to regenerate the vagrant inventory.

To install concourse simply run `ansible-playbook -i /inventory/vagrant configure.yml`.  The vagrant "web" VM will be running concourse ui.  Use `vagrant ssh web -c ifconfig` to figure out it's IP.

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
