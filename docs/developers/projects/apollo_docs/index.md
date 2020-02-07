# apollo_docs

This repository contains the version-controlled markdown used by the `mkdocs` python library to build the documentation site you are currently visting. It is deployed as part of the `apollo_docs' role, which requires the `common` and `nginxinc.nginx` roles as pre-requisites.

Editing the site is quite simple; see the documentation for `mkdocs` for a tutorial.

## Troubleshooting: 

If you've updated the markdown in the main repository (note that this is separate from the role), commited it to github, and you're still not seeing the changes reflected on the live site after using the ansible role, make sure you ran `mkdocs build` in order to actually *build* the site.


