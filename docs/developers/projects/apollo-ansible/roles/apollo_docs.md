# apollo_docs

This role is used to build the site you're currently visiting. It requires the `common` and `nginxinc.nginx` roles to be applied before hand.

The site is built using the `mkdocs` python library, a static site generator that uses markdown files. See the documentation for mkdocs for a tutorial. Its quite simple.

## Deployment Notes

The role pulls down a copy of the site from github using a key. The nginx document root is set as `/usr/share/nginx/html/apollo_docs`. The proper SELinux context is set on the document root, so you shouldn't have any problems. 

As of 2020-02-07, the actual configuration of the virtual host in NGINX happens during the deployment of the `nginxinc.nginx` role. 

## Troubleshooting 

If you've updated the markdown in the main repository (note that this is separate from the ansible role), commited it to github, and you're still not seeing the changes reflected on the live site, make sure you ran `mkdocs build` in order to actually *build* the site.


