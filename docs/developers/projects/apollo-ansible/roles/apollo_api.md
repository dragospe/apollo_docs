# apollo_api

This role configures and deploys the main flask app used for pulling down Garmin API data and storing it in the database. 

## Prerequisites 
The `nginxinc.nginx` and `common` roles should be applied first. The reverse-proxying setup between gunicorn and nginx should occur in the `nginxinc.nginx` role. 

## Overview

This role pulls down the `apollo_flask` repository from bitbucket using a private key and installs the app in the directory `/usr/lib/apollo_site` by default (see the *Variables* section below.) The application is installed in a python virtual environment.

The application requires a valid `instance/config.py` file to pull configuration data from. This is where API keys, the project name, and flask secrets are set. *Keep sensitive information valuted.* 

The role also tags port 8000 (gunicorn's default) with the `http_port_t` SELinux type. 

The role also sets up and enables a systemd service.

## Variables

Role specific variables:

| name | usage | default |
|:-----|:------|:-----|
| `apollo_flask_git_uri` | The URI used to pull down a git repo. You may need to change this to switch to a development branch. | `ssh://git@bitbucket.org/pdragos1/apollo_flask.git`
| `apollo_flask_git_version` | The version used to pull down the repository | `HEAD`
| `apollo_flask_application_directory` | The directory into which the flask app will be installed. | `/usr/lib/apollo_site`
| `apollo_flask_log_directory` | The directory in which gunicorn will keep its logs | `/var/log/apollo_site`
