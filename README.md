# Ansible Collection - adfinis.openxchange

Deploy [OX App Suite](https://www.open-xchange.com/products/ox-app-suite/) and some of its components.

## Requirements

- This collection is designed to work with Debian 10 or later.
    - The exact version of Debian that will work depends on the version of OX App Suite being deployed.
- A MySQL/MariaDB server or Galera cluster must be provided and be reachable from both the Ansible machine and its targets.
    - OX App Suite requires multiple schemas (databases) within the database server.  This collection can take care of creating them.

## Dependencies

- `community.general`
- `community.mysql`

## License

See [LICENSE](LICENSE)

## Author Information

The `adfinis.openxchange` collection was written by:

- [Adfinis AG](https://www.adfinis.com/) | [GitHub](https://github.com/adfinis)


## Usage

### The Bare Minimum Single Node

In this example, we'll set up a single node OX App Suite instance.

First, let's write a small playbook:

```yaml
---

- hosts: oxappsuite
  roles:
    - adfinis.openxchange.appsuite
    - adfinis.openxchange.apache2

```

Next, we'll need quite a lot of hostvars to configure our instance (make sure to protect passwords and other secrets with e.g. ansible-vault in a real-world deployment).

First, let's take care of the settings for App Suite:

```yaml
---
# The connection information for the database
appsuite_configdb_host: sql.example.org
appsuite_configdb_port: 3306

# The password for the openxchange database user to create
appsuite_configdb_pass: supersecurecdbpassword

# Credentials for an administrative user that can create new schemas and users
appsuite_configdb_root_user: root
appsuite_configdb_root_pass: supersecurerootpassword

# Password for the global "oxadminmaster" admin user - will be created automatically when the cluster is initialized
appsuite_master_pass: supersecureadminpassword

# The path where user data are stored.  On a multi node cluster, this must be the same storage on all nodes, e.g. a shared NFS mount.
appsuite_filestore_path: /var/opt/filestore

# These use localhost by default
appsuite_global_imapserver: "imap://imap.example.org:143"
appsuite_global_smtpserver: "smtp://smtp.example.org:25"

# Some features require a subscription.  Can be omitted if you don't have an OX App Suite licence key.
appsuite_license_key: "286c0282-e0ca-481a-9a98-02eb55222dcd"

# The salt used for cryptographic hashes in session cookies.  Generate your own!
appsuite_session_cookie_salt: GenerateYourOwnSecretForUseAsASalt

# The secret for symmetric encryption of session data.  Generate your own!
appsuite_session_encryption_key: GenerateYourOwnSecretForSessionDataEncrypyion

# Credentials for the management REST API admin user
appsuite_restapi_basicauth_user: oxrestapi
appsuite_restapi_basicauth_password: supersecurerestpassword
```
 
Now we also need an Apache2 reverse proxy in front of App Suite:

```yaml

apache2_vhost_servername: ox.example.org

# We should be using HTTPS!  Deploy your own certs and enter the paths here:
apache2_ssl_cert_filename: /etc/ssl/certs/ox.example.org.crt
apache2_ssl_cert_filename: /etc/ssl/private/ox.example.org.key

# Redirect from :80 to :443
apache2_ssl_vhost_redirect_enabled: true

# Enable this only once the redirect has been tested successfully
#apache2_ssl_vhost_redirect_permanent: true
```

Now it's time to deploy.  The first time deployment must be done a bit differently than future config changes, as the databases must be initialized first:

```shell-session
$ ansible-playbook playbook.yml --limit oxappsuite --tags role::appsuite:install,appsuite:init --diff
$ ansible-playbook playbook.yml --limit oxappsuite --diff
```

In future playbook runs (to roll out config changes or upgrade the cluster), omit the tagged run:

```shell-session
$ ansible-playbook playbook.yml --limit oxappsuite --diff
```

If everything went right, you should now have OX App Suite running at `https://ox.example.org/`.


### Multi Node Cluster

If you want to deploy an App Suite cluster, a few more options are required.

First, let's make sure our playbook only ever touches a single node at once:

```yaml
---

- hosts: oxappsuite
  serial: 1  # This is the magic bit
  roles:
    - adfinis.openxchange.appsuite
    - adfinis.openxchange.apache2

```

The `adfinis.openxchange.appsuite` role's restart handler takes care of waiting until one node is fully operational again before advancing to the next node.

Now we need to provide some additional config options.  **All options from the minimal example above are still needed, the following are an addition**:

```yaml
---

# This is the CLUSTER name (not SERVER name) that's registered in the configdb.  This must be the same across all nodes that are part of the same cluster.
appsuite_servername: example

# The name of the Hazelcast cluster set up between the App Suite nodes:
appsuite_cluster_name: example

# By default, App Suite binds to localhost only.  In a cluster setup, it needs to bind to an interface where it can reach the other cluster members.
appsuite_network_listener_host: "*"
# Provide the IP addresses of all the cluster nodes (including each node's own):
appsuite_hazelcast_network_join: static
appsuite_hazelcast_network_static_nodes:
  - "192.0.2.11"
  - "192.0.2.12"
  - "192.0.2.13"

# This option must be different between the App Suite nodes (e.g. APP1, APP2, ...).
# It's set as part of the session cookie and is used for routing the user to the correct node.
appsuite_jkroute: APP1


# Apache can redirect sessions to the correct App Suite node, however, this should not happen with a properly configured load balancer.
# Enter the name of an inventory group here that contains all nodes of the cluster (and only those), and the jkroutes will be looked up from the hostvars.
apache2_balancer_hostgroup: oxappsuite
```

When deploying for the first time, we again need to make that the database is only initialized once:

```shell-session
$ ansible-playbook playbook.yml --limit ox1.example.org --tags role::appsuite:install,role::appsuite:init --diff
$ ansible-playbook playbook.yml --limit ox2.example.org,ox3.example.org --tags role::appsuite:install,role::appsuite:oxinstaller --diff
$ ansible-playbook playbook.yml --limit oxappsuite --diff
```

Note the difference between the first node (where `role::appsuite:init` is applied) and the remaining nodes (where only the `role::appsuite:oxinstaller` part is applied)!

There is one final thing that needs to be taken care of when deploying a multi node cluster:

There needs to be a load balancer in front of the App Suite nodes.
The load balancer should be aware of the strings configured in `appsuite_jkroute` and which node they correspond to.
If such a string occurs in part of the `JSESSIONID` HTTP cookie, the HTTP request should be routed to the appropriate node.

The setup of this load balancer is out of the scope of this Ansible collection.

