# Vault Installation

### Download, Unpack and copy the binary into place.

```
$ export VAULT_VERSION=1.2.3 # latest at the time of writing
$ curl -LO https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip
$ unzip -q vault_${VAULT_VERSION}_linux_amd64.zip
$ sudo cp vault /usr/local/bin/
```

There’s not a lot that we need to do with Vault’s configuration file.
Many of the important configuration items are stored in the encrypted backend.

Let’s add a minimal (i.e. non-production) configuration:
Note that i am turning off ssl on the vault server. I ended up using the EC2 load balancing service
to serve my vault out over ssl.

```
$ mkdir /etc/vault.d
$ cat > /etc/vault.d/vault.hcl <<EOF
storage "file" {
  path    = "/tmp/vault"
}
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}
ui = true
EOF
```

Start the vault server

```$ vault server -config=/etc/vault.d/vault.hcl```

### Initialization

Once Vault is up and running, it must be initialized and unsealed. 
Since Vault is running in the foreground, we’ll need to jump over to another terminal window.
```
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ echo 'export VAULT_ADDR="http://127.0.0.1:8200"' >> ~/.bashrc
$ export PATH=$PATH:/usr/local/bin
```
Now, we should be able to get the status of the server:
```
$ vault status
Error checking seal status: Error making API request. URL: GET http://127.0.0.1:8200/v1/sys/seal-status
Code: 400. Errors:

* server is not yet initialized
```

Next, we initialize Vault:

```
$ vault operator init -key-shares=1 -key-threshold=1
Unseal Key 1: ySEWQMzGk3l6p+u2xkpjxL+BLGIz8/vauk8NmgvmCx0=

Initial Root Token: 27dd03e7-8cda-0e5f-d53a-d64196945ab9

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 1 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault rekey" for more information.
```

# Unseal

Here you will need to login first using the token output during the unseal operation above.
Then, you will need to enter the unseal key to validate the unseal operation.

```
$ vault login
$ vault operator unseal
Unseal Key (will be hidden):
Key             Value
---             -----
Seal Type       shamir
Sealed          false
Total Shares    1
Threshold       1
Version         0.10.1
Cluster Name    vault-cluster-0241c38f
Cluster ID      5b9ab5c2-fd50-46d8-c016-3411c04570bc
HA Enabled      false
```

## Auditing - enable syslogging for credential requests
```
$ vault audit enable syslog
Success! Enabled the syslog audit device at: syslog/
```

## SSH engine - enable the ssh secrets engine inside vault on /ssh-client path
```
$ vault secrets enable -path=ssh-client ssh
Success! Enabled the ssh secrets engine at: ssh-client/
```

## Configure certificate authority

We redirect stdout to a file because we will ultimately need the generated certificate on all client systems for verification

```
vault write \
   -field=public_key \
   ssh-client/config/ca \
   generate_signing_key=true | sudo tee /etc/ssh/trusted-user-ca-keys.pem
```

We can then configure the ssh daemon on client machines to to use our new CA certificate

```
$ echo "TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem" | sudo tee -a /etc/ssh/sshd_config
$ sudo sshd -t # test config just to be safe - no output means all is ok
$ sudo systemctl reload sshd
```

Now, lets add a user to the system for later testing, this user may already exist locally or in a directory. Im using richard locally for now.
```
$ sudo useradd richard
```

## Add Roles to vault

Roles in Vault are created by writing data to special paths.
Roles provide a fine-grained interface for constraining the details that go into a signed client certificate.
For example, one could have a role that allows SSH access as the root user to non-prod IP ranges and another role to SSH as root to production.

### Create the role
```
$ cat > regular-user-role.hcl <<EOF
{
    "allow_user_certificates": true,
    "allowed_users": "richard,ec2-user",
    "default_user": "richard",
    "allow_user_key_ids": "true",
    "default_extensions": [
        {
          "permit-pty": ""
        }
    ],
    "key_type": "ca",
    "ttl": "15m0s",
    "allow_user_key_ids": "false",
    "key_id_format": "{{token_display_name}}"
}
EOF
```

# Now we write the role to vault
```
$ cat regular-user-role.hcl | vault write ssh-client/roles/regular -
Success! Data written to: ssh-client/roles/regular
```

## Policies

Policies allow us to define which users can request certificates from (i.e. write data to) which paths. Generally paths determine the level of access but we are keeping it basic for this demo.

```
$ cat > regular-user-role-policy.hcl <<EOF
path "ssh-client/sign/regular" {
    capabilities = ["create","update"]
}
EOF
```

### create the policy
```vault policy write ssh-regular-user regular-user-role-policy.hcl```

## Client Workflow

In order to test the client workflow for the regular user path,
we’ll need a users. We start by enabling the userpass authentication method:
```
$ vault auth enable -path=plain userpass
$ vault write auth/plain/users/richard password="foobar" policies="ssh-regular-user"
```
### Now, we’re ready to test the client-side workflow.
```
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ ssh-keygen -qf $HOME/.ssh/id_rsa -t rsa -N ""

$ vault login \
    -path=plain \
    -method=userpass \
    username=richard \
    password=foobar

$ vault write \
    -field=signed_key \
    ssh-client/sign/regular \
    valid_principals="richard" \
    public_key=@$HOME/.ssh/id_rsa.pub \
    > $HOME/.ssh/cert-signed.pub

$ ssh-keygen -Lf $HOME/.ssh/cert-signed.pub
```

# make vault a service
vi /etc/systemd/system/vault.service


```
[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=60
StartLimitIntervalSec=60
StartLimitBurst=3
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
```

# Start Vault
```
sudo systemctl enable vault
sudo systemctl start vault
sudo systemctl status vault
```

you are now ready to start consuming the vault from Tower.


