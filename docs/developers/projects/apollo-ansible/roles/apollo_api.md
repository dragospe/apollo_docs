# apollo_api

### Variables

Role specific variables:

| name | usage | default |
|:-----|:------|:-----|
| `apollo_flask_git_uri` | The URI used to pull down a git repo. You may need to change this to switch to a development branch. | `ssh://git@bitbucket.org/pdragos1/apollo_flask.git`
| `apollo_flask_git_version` | The version used to pull down the repository | `HEAD`
| `apollo_flask_application_directory` | The directory into which the flask app will be installed. | `/usr/lib/apollo_site`
| `apollo_flask_log_directory` | The directory in which gunicorn will keep its logs | `/var/log/apollo_site`

