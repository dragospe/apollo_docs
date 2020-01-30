# apollo-ansible
Your one-stop-shop for deployment playbooks and info. Click [here](#role-overview) to see details about a specific role.

## Infrastructure Overview 
The current deployment is across two virtual servers from URMC's academic IT. It doesn't appear that we can reimage these, so don't fuck up.

The first server is: 
    `apollosql.urmc-sh.rocheter.edu`
  which houses:
   * a postgresql database cluster, with a database to store Garmin data and a database grafana uses for its backend
   * grafana
   * an nginx server to proxy requests to grafana and serve the static documentation site.
 
 The second server is:
  `apolloweb.urmc-sh.rochester.edu`
 which houses an nginx server that proxies requests to gunicorn/flask. 

## Network Overview
The below information is current as of 2020-01-30:
### apollosql
 * Name: `apolloweb.urmc-sh.rochester.edu`
 * IPv4 Address (Static): `172.168.139.232/22`
 * Gateway: `172.16.139.250`
 * DNS Servers: `172.17.254.30; 172.17.254.130`
 * Firewalld services: `dhcpv6-client http https postgresql ssh`
 * Nginx:
	 * listens on port 80, all names/addresses for redirects to https
	 * Listens on all addresses, port 443, on names:
		 * `docs.apollo-af.org` (served statically)
		 * `dash.apollo-af.org` (proxied to `/grafana_socket/grafana.socket`)
 * Postgresql:
	 * Listens on port 5432, all addresses. The user `apollo` is the only user allowed to connect over the network, and only to the `apollo-site` database. md5 authentication is used
	 * Listens on unix socket. The `grafana` and `postgres` users both use peer authentication.

### apolloweb
__NOTE: This server obtains its network info from DHCP!!__ It might change. We don't know.
 * Name: `apolloweb.urmc-sh.rochester.edu`
 * IPv4 Address (Static): `10.221.3.133/24`
 * Gateway: `10.221.3.250`
 * DNS Servers: `172.17.254.30; 172.17.254.130`
 * Firewalld services: `dhcpv6-client http https ssh`
 * Nginx:
	 * listens on port 80, all names/addresses for redirects to https
	 * Listens on all addresses, port 443, on name `auth.apollo-af.org`:
		 * Location `/` is proxied to gunicorn, listening on localhost port 8000
		 * Location `/api_client/garmin/` is proxied to gunicorn on localhost, port 8000.   Garmin sends data using the following IPs, which are whitelisted:
			 * 98.100.124.0/24
			 * 98.100.125.0/24
			 * 204.77.162.0/24
			 * 204.77.163.0/24

## Users and groups
The following users are added in the `common` role and should exist on all hosts. They are handled individually by the ansible scripts in the `tasks/user_accounts/` directory. Currently the users that should exist on all hosts are:
 * apage2 (provided by Active Directory in the `urmc-sh.rochester.edu` domain)
 * pdragos (provided by Active Directory in the `urmc-sh.rochester.edu` domain)
 * apollo (local user)

On `apollosql`, the users `nginx` and `grafana` (installed by their respective applications) should both be part of the `grafana_socket` group. This is necessary to allow nginx and grafana to communicate over a unix socket in a SGID directory.

## Crypto and Security Overview
Passwords and keys should be stored in a password manager or ansible-vault'ed. 
 * Postgresql communicates over TLS with self-signed cert. Authentication is either local (`peer`) or md5. 
 * Grafana seems to require a plaintext database password in its configuration file. Care should be taken to ensure that the proper permissions are set on `/etc/grafana/grafana.ini`.
 * Nginx has individual TLS certificates for all host names it is responsible for. As of 2020-01-30, we do not have public IPs for the virtual machines; when we do, certbot will be responsible for maintaining these certificates.
 * A restrictive umask is in place. Be aware of this.
 * SELinux should work in enforcing mode.

## Deployment
To get running quickly:
* Set up your inventory file at `hosts/prod/inventory`. You'll need an `apolloweb` host and an `apollosql` host.
* Run ```ansible-playbook -K deploy_to_prod.yml --vault-password-file /path/to/vault_pass -i hosts/prod/inventory```
### Lifecycle Environments


This repository is designed to configure development (`dev`), testing (`test`), and production (`prod`) lifecycle enviornments. To do so, three different inventory files in their respective directories under the `hosts/[lifecycle state]/` directory. 

Variables specific to each lifecycle state may be set in accordance with inventory-based variables ([`group_vars`](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#assigning-a-variable-to-many-machines-group-variables)  , [`host_vars`](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#assigning-a-variable-to-one-machine-host-variables), etc.)

*Note:* In ansible, `group_vars` take precedence over variables in `roles/[rolename]/defaults/main.yml`. 

### Playbook flow (current as of 2020-01-30):
 * set up all servers (`apolloweb` and `apollosql` with the common role
 * cetup postgresql
 * configure postgres
 * do a hacky work-around to get SSL working
 * set up nginx
 * open ports
 * set up the webapp
 * set up grafana
 * set up the documentation site
### Variables:
Where to find them: 

*  `hosts/[lifecycle state]/group_vars` sets playbook-wide variables. This should be the primary source for configuration parameters.
*  The playbook itself. This is where some more verbose, static parameters should be set (e.g., `pg_hba.conf` or the `nginx.conf` entries. 

Look at the relevant files to get an idea of the variables you need to set.

### Variables
Variables that relate to site-wide or lifecycle-wide configuration are listed below.

Complete descriptions of *role-specific* variables (required or optional) may be found in the respective `README.md` files. 

The variables listed below are *generally* defined in `hosts/[lifecycle]/group_vars/all.yml`.

|Variable      | Description        | Example                                                                   |
|-------------|-------------------------------------------------------------------------------------------------------------------|--------------|
|lifecycle_state| Either 'dev', 'test', or 'prod', depending on the inventory file used. | prod
|apollo_site_database_name | The name of the database the main API site will use | apollo_site
|apollo_site_database_user | The username that SQLalchemy/flask will connect as | apollo
apollo_site_database_password | a vaulted password to authenticate the postgresql user for the main API site | `!vault | $ANSIBLE_VAULT;1.1;AES256         36643264643431346165613138326261356335326265353337363762663836653566336135303366      343365366...`
apollo_site_database_connection_cidr_ip | The IPv4 address that the apollo site will connect to its backend database from, for whitelisting purposes | `192.168.122.191/24`
| apollo_site_database_uri | The URI that the SQLalchemy should use to connect to the database | `"postgresql://{{ apollo_site_database_user }}:{{ apollo_site_database_password }}@apollosql.urmc-sh.rochester.edu:5432/{{ apollo_site_database_name }}"`
| apollo_monitor_domain | The domain name (for grafana) | `dash.apollo-af.org`
| apollo_monitor_root_url | The *full* URL used for the site. Needed for redirects.  | `https://dash.apollo-af.org/`
| apollo_monitor_postgresql_backend_host | Controls how grafana accesses its backend database | `/run/postgresql/`
| apollo_monitor_postgresql_backend_database_name | The name of the database grafana uses as its backend| `grafana`
| apollo_monitor_postgresql_backend_database_username | The username grafana connects to its backend database with | `grafana`
| apollo_monitor_postgresql_backend_database_password| The vaulted password grafana uses to connect to its backend database| `!vault |$ANSIBLE_VAULT;1.1;AES2560  353035646165360356233370...`
| apollo_api_nginx_server_name | The name that nginx listens on to proxy to the main site | `auth.apollo-af.org`
| apollo_docs_nginx_server_name | The name that nginx listens on to proxy to the documentation site | `docs.apollo-af.org`
| apollo_monitor_nginx_server_name | The name that nginx listens on to proxy to grafana | `dash.apollo-af.org`


## Role Overview


| Role        | Description                               | Pre-requisites | 
|-------------|---------------------------------------------------------------------------------------------------------------------------------|----------|
|[apollo_api](roles/apollo_api) | Sets up flask and gunicorn to serve the main site | `common, nginxinc.nginx`
|[apollo_docs](roles/apollo_docs)| Sets up the static mkdocs documentation site | `common, nginxinc.nginx`
|[apollo_monitor](roles/apollo_monitor)  | Configures a host to perform monitoring functions (grafana).  | `common, nginxinc.nginx`. Requires a a backend database, so *some* host must have the `geerlingguy.postgresql` and `apollo_pg` roles applied as well                  |
| [apollo_pg](roles/apollo_pg)   | Configures a host running postgreSQL to function as apollo's long-term data store. Note: the `geerlingguy.postgresql` role must already be                                            | `common, geerlingguy.postgresql`
| [apollo_scripts](roles/apollo_scripts)  | Configures a host to run scripts related to geneACTIV, medtronic, REDCap, etc. | N/A                                                        
| [common](roles/common)      | Common configuration applied to all hosts, including common software, user accounts, security hardening, etc. | N/A
| geerlingguy.postgresql | Configures a host with a postgresql server. Pulled from ansible-galaxy; do not modify. | N/A
  | nginxinc.nginx | Configures a host with an nginx server. Pulled from ansible-galaxy; do not modify  | N/A


