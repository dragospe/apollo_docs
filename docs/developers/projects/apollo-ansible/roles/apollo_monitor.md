apollo_monitor
=========

Synopsis
-----------
This role sets up the monitoring infrastructure (grafana) for the apollo project. It currently sets up grafana. Out of the box, this role will provision datasources, dashboards, and users, as well as configure the server. It will set up some user permissions. 

**Note:** To completely restore an apollo_monitor instance, you must copy the `grafana.db` file and restore the appropriate configuration. Back this file up as necessary. A separate playbook will be developed to dump the `grafana.db` file to an S3 bucket.

READ THIS: Development Notes 
--------------------------------------------
 * DNS is not currently setup, especially not for the dev branch.  A play may eventually be developed to handle the following (depending on how our PKI plays out), but in order to get TLS working with the appropriate security checks in place, you will have to:
	 * On the server:
	 	 * Add `apollo_monitor.dev` to the hosts file. Binding it to `127.0.0.1` is fine (for now), since grafana set up to listening on all interfaces 
		 * Add the certificate authority used to sign the TLS dev certificate. This can be found in `hosts/dev/files/`. Copy it `/etc/pki/ca-trust/source/anchors/`.
		 * Run `update-ca-trust`
	 * On the client:
		 * Add `apollo_monitor.dev` to the hosts file with the IP address of the server.
		 * Add the certificate authority used to sign the TLS dev certificate (as above, for Fedora; or your distribution's equivalent).
		 * Run `update-ca-trust` (for Fedora, for your distribution's equivalent).
 
Requirements
------------
This role targets a CentOS 7 minimal installation with the apollo-af `common` role applied as a pre-requisite.

Adding Users
----------------
To add a user, add a `[user]_account.yml` file to the `tasks/user_accounts/` directory with the appropriate variables filled out. See `tasks/user_accounts/template.yml` for an example. 

Users are added using the API grafana provides. This can get clunky: certian actions return error codes that can be hard to parse in ansible (for example, a 404 is returned when a non-existant user is queried.)

Files
-----------
 The role expects two files and two folders in the `files/` directory:

  * The `[env]_apollo_monitor_server.crt` file, containing the appropriate (signed) certificate to use for TLS.
  * The (vaulted) `[env]_apollo_monitor_server.key` file, containing the corresponding private key.
  * The `datasources/` directory, containing:
	  * `*.yml` files describing datasources to be provisioned by the server
  * The `dashboards/` directory, containing
	  * `*.yml` files describing dashboard providers
	  * `*.json` files describing dashboards to be provisioned
	 
 See the [grafana docs](https://grafana.com/docs/administration/provisioning/#example-datasource-config-file) for more information on provisioning.
 
Variables
-----------
Global, user-defined variables are listed below. See the `template.yml` file described in the "Adding Users" section for variables related to adding users.

| Variable:                            	| Choices/Defaults:                                                                                                                                                                	| Declared in:                                                                      	| Comments:                                                           	|
|--------------------------------------	|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|-----------------------------------------------------------------------------------	|---------------------------------------------------------------------	|
| apollo_monitor_grafana_admin_password | *Format*: A vaulted plaintext file containing the "admin" account password. | `defaults/main.yml	`|
| apollo_monitor_domain | *Format*: FQDN of the grafana server | `../../hosts/[env]/group_vars/apollo_monitor/apollo_monitor.yml` | The public facing domain name used to access grafana from a browser
| apollo_monitor_root_url | *Format*: URL| `../../hosts/[env]/group_vars/apollo_monitor/apollo_monitor.yml` | The full public facing url you use in browser, used for redirects and emails|. 
| apollo_monitor_postgresql_backend_host | Hostname or IP | `../../hosts/[env]/group_vars/apollo_monitor/apollo_monitor_postgresql_backend_details.yml` | The host name for grafana to connect to for it's backend database. |
| apollo_monitor_postgresql_backend_database_name | *Format*: String |   `../../hosts/[env]/group_vars/apollo_monitor/apollo_monitor_postgresql_backend_details.yml` | Name of the backend database to connect to |
| apollo_monitor_postgresql_backend_database_username | *Format*: String |   `../../hosts/[env]/group_vars/apollo_monitor/apollo_monitor_postgresql_backend_details.yml` | Username of the database role grafana should connect as for it's backend. | 
| apollo_monitor_postgresql_backend_database_password | *Format*: Vaulted string |   `../../hosts/[env]/group_vars/apollo_monitor/apollo_monitor_postgresql_backend_details.yml` | Vaulted password of the database role grafana should connect as for it's backend. |

Security Considerations
-----------
**TODO**: 
* evaluate if the vaulted password for apollo_monitor_grafana_admin_password  is getting logged anywhere (ansible or grafana)
* Evaluate if grafana is storing passwords (user, datasource, etc in plaintext)

This role sets up TLS and sets some options in `grafana.ini` to ensure security. Grafana has built-in brute-force throttling, so fail2ban has not been configured. 

As of this writing (2019-06-26), it appears that certain API calls must use Basic Auth (read: plaintext) to authenticate (rather than API tokens). This is sub-optimal, but certain tasks can only be automated in this way. Ensure that TLS is working!

Reading through various bug reports, I have concluded that security is not a first-class concern of the grafana developers, although it is getting worked on. I am uncertain whether or data source passwords are being stored in plaintext.

Network Considerations
-------

Nginx should be configured to listen on port 80,443 on the name `dash.apollo-af.org` and proxy the requests (**TLS only**) to socket that grafana is listening on. 

The default umask for the servers in this project is restrictive. To allow nginx and grafana to communicate, you must:

* Set the selinux context for `"/grafana_socket(/.*)?"` to `system_u:object_r:httpd_var_run_t:s0`
* Create a group `grafana_socket`
* Add users `nginx` and `grafana` to the `grafana_socket` group 
* Restart nginx (so that the new group takes effect)
* Create a folder `/grafana_socket` with owner `root`, group `grafana`, and permissions `2770`. Make sure that the appropriate SELinux context is applied.

