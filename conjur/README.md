# AAM Xperience Workshop

Welcome again to CyberArk AAM Xperience Workshop!   This page contains everything you need for the hands-on session.

## Session Goal 
Today we will install Ansible, a configuration management tool for managing our servers.
At first, we will simply configure Ansible without securing the embedded secrets and discuss the risk involved with storing cleartext credentials.
Then we will then install Conjur and integrate it with Ansible. At the end of the session we will see how the risk of embedded secrets will be eliminated by using Conjur.


## Environment Preparation
If you are running this lab on your own machine and not CyberArk's SkyTap environment, follow below steps to create a new lab envirnment.

1. Install 3 RHEL 8 systems, named `conjur`,`host-1` and `host-2`
2. System Update on all three servers
```
yum update
```

3. Install podman on `conjur` server
```
yum -y install python3 git podman
```

4. Install `podman-compose` & `jq` on `conjur` server 
```
curl -L "https://raw.githubusercontent.com/sheeshkebab/podman-compose-1/devel/podman_compose.py" -o /usr/local/bin/podman-compose

chmod +x /usr/local/bin/podman-compose

alternatives --install /usr/bin/podman-compose podman-compose /usr/local/bin/podman-compose 1

yum -y install jq

```

## Managing the servers using Ansible 
In this section we see how Ansible is used to manage the severs remotely.

1. Let's install Ansible on `conjur` server

Run below commands to install ansible.
```
easy_install pip
pip install ansible
```
ref: https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

2. Create service accounts on `host-1` & `host-2`

Note: The passwords that are set here will be used later on by Ansible.

On `host-1`:
```
sudo useradd -m -d /tmp service01
sudo passwd service01
Enter new UNIX password: W/4m=cS6QSZSc*nd
Retype new UNIX password: W/4m=cS6QSZSc*nd
```


On `host-2`:
```
sudo useradd -m -d /tmp service02
sudo passwd service02
Enter new UNIX password: 5;LF+J4Rfqds:DZ8
Retype new UNIX password: 5;LF+J4Rfqds:DZ8
```

3. Create `inventory` & `playbook.yml` on `conjur` host

Create an inventory file
```
vi inventory
```

Insert below content in the `inventory` file. Note that we are storing credentials in cleartext format in a config file.

```
[db_servers]
host-1 ansible_connection=ssh ansible_ssh_user=service01 ansible_ssh_pass=W/4m=cS6QSZSc*nd
host-2 ansible_connection=ssh ansible_ssh_user=service02 ansible_ssh_pass=5;LF+J4Rfqds:DZ8
```


Create a playbook, called `playbook.yml`. 
```
vi playbook.yml
```

Insert below content in the `playbook.yml` file. This playbook finds the username and hostname on the target machine and print son the screen.

```
- hosts: db_servers
  tasks:
    - name: Get user name
      shell: whoami
      register: theuser

    - name: Get host name
      shell: hostname
      register: thehost

    - debug: msg="I am {{ theuser.stdout }} at {{ thehost.stdout }}"
```

Disable host key checking by running in command:

```
export ANSIBLE_HOST_KEY_CHECKING=False
````

Let's try running the playbook and test connectivity:

```
ansible-playbook -i inventory playbook.yml
````


## The risk
Looking at what we have just done, think about the questions below and discuss the risks involved.

-	Who can view the secrets stored in the plaintext file?
-	At which part of the process the secrets are used?
-	How can we rotate the secrets?
-	Is this process being audited?


## Securing the Environment

### Install Conjur OSS 

We now install Conjur OSS on `conjur` for securing the secrets.
Refer to the link below for more detail on Conjur OpenSource Software: https://www.conjur.org/get-started/quick-start/oss-environment/

Run these commands to download and setup Conjur OSS. Make sure you run the commands individually and inspect successful completion before moving to the next coommand.

```
git clone https://github.com/cyberark/conjur-quickstart.git
cd conjur-quickstart
podman-compose pull
podman-compose run --no-deps --rm conjur data-key generate | tail -1 > data_key
export CONJUR_DATA_KEY="$(< data_key)"
podman-compose up -d
podman ps -a
podman-compose exec conjur conjurctl account create myConjurAccount > admin_data
podman-compose exec client conjur init -u conjur -a myConjurAccount
```

The admin password can be found in `admin_data` file:

```
cat admin_data
```

Now login as Conjur admin:

```
podman-compose exec client conjur authn login -u admin
```

### Loading Policy & Secrets to Conjur

The first step is to create a top-level policy that defines two empty policies: `db` and` ansible`. Save this policy as `root.yml`:

`vi root.yml`

Add below content to `root.yml`:

```
- !policy
  id: db

- !policy
  id: ansible
```

Save and exit the file.

Then create the `db` policy. This policy will create the target system information.

```
vi db.yml
```


Add below content to `db.yml` file:

```
- &variables
  - !variable host1/host
  - !variable host1/user
  - !variable host1/pass
  - !variable host2/host
  - !variable host2/user
  - !variable host2/pass

- !group secrets-users

- !permit
  resource: *variables
  privileges: [ read, execute ]
  roles: !group secrets-users

# Entitlements 
- !grant
  role: !group secrets-users
  member: !layer /ansible
```
Save and close the file.


We now create the `ansible.yml` to be used to retreive the secrets from Conjur.

```
vi ansible.yml
```

Add below content to `ansible.yml`:

```
- !layer
- !host ansible-01
- !grant
  role: !layer
  member: !host ansible-01
```

Save and exit the file.

We now load all the policies that have just been created to Conjur.

Load `root` policy to conjur:

```
podman cp root.yml conjur_client:/tmp/
podman-compose exec client conjur policy load --replace root /tmp/root.yml
```

Load `ansible` policy to conjur:

```
podman cp ansible.yml conjur_client:/tmp/
podman-compose exec client conjur policy load ansible /tmp/ansible.yml | tee ansible.out

```

Load `db` policy to conjur:

```
podman cp db.yml conjur_client:/tmp/
podman-compose exec client conjur policy load db /tmp/db.yml
```

Our final step is to create secrets and add them to Conjur. These secrets will be used later on by Ansible to connect to the target hosts.

Host 1 host name: 
```
podman-compose exec client conjur variable values add db/host1/host "host-1" 
```

Host 1 user name: 
```
podman-compose exec client conjur variable values add db/host1/user "service01" 
```

Host 1 password: 
```
podman-compose exec client conjur variable values add db/host1/pass "W/4m=cS6QSZSc*nd"
```

Host 2 host name: 
```
podman-compose exec client conjur variable values add db/host2/host "host-2" 
```

Host 2 user name: 
```
podman-compose exec client conjur variable values add db/host2/user "service02" 
```

Host 2 password: 
```
podman-compose exec client conjur variable values add db/host2/pass "5;LF+J4Rfqds:DZ8"
```


### Integrating Ansible

In this section we integrate Ansible with Conjur and see how we can retrieve credentials from Confur and remove the embedded secrets from the Ansible inventory file.

#### Setting up Role & Lookup plugin
The first step is to install a Conjur Ansible role. This is essentially a plugin for Ansible to work with Conjur.

```
ansible-galaxy install cyberark.conjur-lookup-plugin
```


#### Configure SSL and Conjur settings

Download the SSL Certificate from the Conjur service and export to a pem file.
```
openssl s_client -showcerts -connect conjur:8443 < /dev/null 2> /dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > conjur-demo.pem
```

Configure Conjur by exporting a number of environmental variables to the system. Run below commands:

```
export CONJUR_CERT_FILE="$PWD/conjur-demo.pem"
export CONJUR_ACCOUNT="myConjurAccount"
export CONJUR_APPLIANCE_URL="https://localhost:8443/"
export CONJUR_AUTHN_LOGIN="host/ansible/ansible-01"
export CONJUR_AUTHN_API_KEY="$(tail -n +2 ansible.out | jq -r '.created_roles."myConjurAccount:host:ansible/ansible-01".api_key')"
```

### Updating Ansible Inventory & Playbook
We now update the ansible inventory & playbook files that was created earlier and remove the embeded secrets in the inventory file.

#### Update inventory file
Open the inventory file and remove existing content under `[db_servers]`. Add two new lines as per below. Note the hostnames are `host1` and `host2` without a `-`.

```
[db_servers]
host1
host2
```

#### Update playbook.yml
Now open the `playbook.yml` file and replace the existing content with below. Note that we now have added a `lookup` function that retrieves credentials dynamically from Conjur in runtime.

```
- hosts: db_servers
  roles:
    - role: cyberark.conjur-lookup-plugin
  vars:
      ansible_connection: ssh      
      ansible_host: "{{ lookup('retrieve_conjur_variable', 'db/' + inventory_hostname+ '/host') }}"
      ansible_user: "{{ lookup('retrieve_conjur_variable', 'db/' + inventory_hostname+ '/user') }}"
      ansible_ssh_pass: "{{ lookup('retrieve_conjur_variable', 'db/' + inventory_hostname+ '/pass') }}"

  tasks:
    - name: Get user name
      shell: whoami
      register: theuser

    - name: Get host name
      shell: hostname
      register: thehost

    - debug: msg="I am {{ theuser.stdout }} at {{ thehost.stdout }}"
```

#### Run the playbook again
It's now time to run the playbook again and observe that the same results can be achieved without embedded credentials, and by dynamically retrieving secrets from Conjur. 

```
ansible-playbook -i inventory playbook.yml
```

### Cleanup

If you want to try it again, you can preform the following actions to cleanup the environment:

- Delete the files created 
- On `conjur`. execute `podman-compose down` to remove the conjur containers
- On `host-1`, exeucte `sudo userdel service01`
- On `host-2`, exeucte `sudo userdel service02`

