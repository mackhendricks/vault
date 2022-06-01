# Configuring Vault on VM's with RAFT

# Install Vault

```
cp vault /usr/local/bin
mkdir /etc/vault
mkdir -p /var/lib/vault/data
cp <vault_license> /etc/vault/vault.lic
```

```
useradd --system --home /etc/vault --shell /bin/false vault
chown -R vault:vault /etc/vault /var/lib/vault/
```

```
cat <<EOF | sudo tee /etc/systemd/system/vault.service
[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault/config.hcl

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault/config.hcl
ExecReload=/bin/kill --signal HUP 
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitBurst=3
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

```
touch /etc/vault/config.hcl
```

```
cat <<EOF | sudo tee /etc/vault/config.hcl
disable_cache = true
disable_mlock = true
ui = true
listener "tcp" {
   address          = "0.0.0.0:8200"
   cluster_address  = "0.0.0.0:8201"
   tls_disable      = 1
}

seal "pkcs11" {
  lib            = "/opt/cloudhsm/lib/libcloudhsm_pkcs11.so"
  slot           = "1"
  pin            = "vault_user:1234567"
  generate_key   = "true"
  key_label      = "vault"
  hmac_key_label = "vault"
}

storage "raft" {
   path  = "/var/lib/vault/data"
   node_id="vault-01"
 }
api_addr         = "http://0.0.0.0:8200"
cluster_addr     = "http://0.0.0.0:8201"
max_lease_ttl         = "10h"
default_lease_ttl    = "10h"
cluster_name         = "vault"
raw_storage_endpoint     = true
disable_sealwrap     = true
disable_printable_check = true
license_path="/etc/vault/vault.lic"
EOF
```



