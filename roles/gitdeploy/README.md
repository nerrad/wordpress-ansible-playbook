## What is the git deploy role and system?

The gitdeploy role configures the server to utilize a git deploy system for pushing changes to your website.  With a git deploy system:

- A bare git repository is setup on the web server and you can access it from a client that has its ssh public key installed on your web server using the an address in the format: 

```
ssh://{server_web_username}@{server_ip_address}/~/{bare_git_repo_directory_name}
```

So, if the value of your `remote_web_user` was `web`, the server ip address you're deploying to is `155.55.55.55` and you configured the `hubdirectory` variable (see variables section) to `hub`.  Then the address you use for remote in your git client would be:

```
ssh://web@155.55.55.55/~/hub

``` 

The value of a git deploy process is that its wired up with a `post-update` hook, so whenever you push a specific named branch to the "hub" on the server, the hook will automatically trigger pulling in the changes to the `wp-content` folder of the linked WordPress site on the same server. 

Some more details:

- This particular system assumes your entire wp-content directory is under version control.
- It is recommended you turn off plugin updates on the WordPress site under version control using this system so that the repository there does not become unclean (with uncommitted changes) via plugin updates.  
- If you do commit directly to the repo in the `wp-content` directory under version control, be aware that deploys from "hub" will reset the head on the `wp-content` folder to the commit of the branch pushed.  So when you make changes, make sure they are pushed to the hub (the same branch).
- Typically this system is setup where pushing to the staging branch pushes to the linked staging site, and pushing to the master branch pushes to the linked production site.  You can push other branches to hub, but unless they are listened for in the `hub/hooks/post-update` hook, nothing will happen automatically after pushing them.


## Variables used by the role.

This role uses the following custom variables defined within your `vars.yml` file:

| Variable | Value Desciption |
| --------- | -------------- |
| staging.hubdirectory | The value you give here is what will be used for directory name of the bare git repo you push changes for your staging website to.  It will also be used as the name of the remote setup within your WordPress wp-content folder. |
| staging.branch_deployed | The value you give here is what branch in the hub's repository will be linked with whatever site is setup for`staging_domain`.
| production.hubdirectory | Same as `staging.hubdirectory` except whatever is setup as the directory here will be linked to the website defined by `main_domain`.  Typically you'll have `hubdirectory` be the same value for both production and staging variations (but with just different branches) but the configuration does allow for completely different repositories if that's your need. |
| production.branch_deployed | The value you give here is what branch in the defined hub's repository that will be linked to whatever site is setup for `main_domain`  


## Role Tasks

This role does the following on the server:

- Checks if the server already has a `post-update` hook setup in the defined hub directory being setup.  If it does, then most of the following steps will be skipped.
- If it doesn't already exist, the defined name for the `hub` directory is created and a bare git repository initialized in it.
- If it doesn't already exist, a post-update file (and contents) will be created in `~/{hub}/hooks/`. 
- This step is the only step that runs regardless of whether the `post-update` hook already exists, it will add to the `post_update` file a new partial script for the web site being setup.  This is so you can add additional site's to the post-update script via ansible subsequently to initial setup.
- If `post-update` was newly created, then the `wp-content` folder for the linked web site is deleted, and recreated (empty).  Then the new 'hub' is setup as a git remote within the `wp-content` folder.

## After running the playbook for this role:

- Add the hub as a remote on whatever machine you use for developing your website.
- Push the changes on a named branch to the remote linked to the website you want to deploy to.