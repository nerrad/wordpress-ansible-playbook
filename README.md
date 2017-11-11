# WordPress Ansible Playbook 

This repository contains an ansible playbook for provisioning a WordPress based server with both a production and staging website, optional ssl certificates (provided free via letsencrypt), PHP 7.1, Mariadb, wp-cli, and nginx.

## Roles

The following roles are in this playbook.

| Role | Description | Tags |
|-------|------------|------|
| common | Installs latest for `fail2ban`, `git-core`, `letsencrypt`,  `unzip`, `python-pip`, `python-dev` | init
| user | Creates a web user and web user group and adds a provided .ssh key.  This user will be used for all web services (and web site folders) | init
| ssh |  Disables root login and password authentication for ssh (hardening) | init |
| nginx | Sets up all the nginx config from the provided repository for given domains. While this role does setup both ssl and non-ssl configs for the provided domain(s), it only _activates_ the non-ssl config.  Also sets up the catchall `default` config for any non request matches to the server. | init, web, production/staging, nginx |
| php | Installs PHP 7.1 and all necessary php extensions WordPress needs.  Also modifies the php configs to set the web user etc. | init |
| mariadb | Installs `mariadb-server` and `python-mysqldb`, sets up root password for all root accounts (generated server side), creates .my.cnf with root password creds (to `/root/.my.cnf` on the server), deletes anonymous mysql server user for server hostname and for local host and removes the mysql test database. | init |
| wp-cli | Installs wp-cli | init
| wordpress | Installs and sets up wp sites for given domains.  You will end up with working vanilla WordPress sites for those domains on the server | init, web, production/staging, wordpress |
| letsencrypt | This will create, authorize, retrieve and setup ssl certificate for the given domain(s) using the letsencrypt service.  It will deactivate the nonssl nginx configuration for the domain and activate the ssl configuration.  It will also setup a cron job for automatically renewing the certificate(s). | init, web, ssl, production/staging |
| gitdeploy | Set's up a git deploy system on the server for pushing changes to your site(s) via git.  [Read more details here.](roles/gitdeploy/README.md) |

## Usage

To get started, you just need to do the following:

### 1. Create a server that has a sudo user already set up.

Make sure that whatever machine you are connecting from as the ansible host has its public key setup on any hosts you are deploying to.

It goes without saying that this is an ansible playbook so its assumed there's already some familiarity with how that works.

### 2. Copy `hosts.example` to `hosts` and add your server info. 

### 3. Copy `vars-sample.yml` to `vars.yml` and add your own values for the variables.

> **Note:** Currently the default `nginx_config_repo` will _not_ work as is until [submitted](https://github.com/A5hleyRich/wordpress-nginx/pull/14) [pull requests](https://github.com/A5hleyRich/wordpress-nginx/pull/13) for that repository are merged in. In the meantime, you can use https://github.com/nerrad/wordpress-nginx.git and the `test-merge-le-phpup` branch there.

These variables are used throughout the playbook.  Here's some more details on them:

| variable | description |
|----------|--------------|
| `main_domain` | Enter what is the domain for the "production" web-site on your server.  The `production` tag will always relate to this domain. |
| `staging_domain` | Enter what is the domain for the "staging" web-site on your server.  The `staging` tag will always relate to this domain. |
| `remote_sudo_user` | This should correspond with the sudo user you created on the host(s) you are deploying to. |
| `remote_web_user` | This is the name of the user that will be created for all web services on your server (nginx, php, and website ownership). |
| `remote_web_user_password` | Enter the encrypted value for the password you'd like assigned to the `remotye_web_user` when its created.  Password authentication for accessing the server (via ssh) as this user is turned off, however its still good practice to set a password on the user.  You can visit [some docs](http://docs.ansible.com/ansible/latest/faq.html#how-do-i-generate-crypted-passwords-for-the-user-module) for instructions on how to encrypt your password for adding here. |
| `web_user_ssh_public_key` | Replace `~/.ssh/id_rsa.pub` if your public ssh key is located somewhere else.  This path is for the machine ansible is being run from. |
| `nginx_config_repo` | In most cases you can just leave this repository address alone, but if you want to use a customized version you can enter your own repository here.  Keep in mind the nginx role heavily relies on this repository so if you *do* customize this, you will likely need to modify the nginx role for the play accordingly. |
| `nginx_config_repo_version` | If there's a specific branch other than master for the repository given with the `nginx_config_repo` variable, then you can indicate that branch here. |
| `main_wp_admin_user` | This value will be used for the production WordPress site admin user when that site is setup. |
| `main_wp_pass` | This value will be used for the production site WordPress admin user password when that site is setup. |
| `main_wp_admin_email` | This is the email address that will be assigned to the production site's admin user. |
| `staging_wp_admin_user` | This value will be used for the staging WordPress site's admin user when that site is created. |
| `staging_wp__pass` | This value will be used for the staging site WordPress admin user password when that site is created. |
| `staging_wp_email` | This value is the email address that will be assigned to the staging sites's admin user. |
|`main_wp_db_name` | This value is what you want the name of the database to be for the production WordPress site. A random password will be automatically generated by ansible on the host |
| `staging_wp_db_name` | This value is what you want the name of the database to be for the staging WordPress site. A random password will be automatically generated by ansible on the host |
| `main_wp_site_title` | What you want used as the title for the production WordPress site when it is created. |
| `staging_wp_site_title` | What you want used as the title for the staging WordPress site when it is created. |
| `letsencrypt_email` | The email address to use when registering/creating letsencrypt ssl certificates.
| `wp_db_import` | This is documented more in the `vars-sample.yml` file.  If you uncomment this configuration and customize it in your `vars.yml` file, then, include a sql file in the `dbimports` directly, then this role will import the db to the designated sites.
| `git_deploy` | This documented more in the `vars-sample.yml` file. I fyou uncomment this configuration and customize it in your `vars.yml` file, this role will configure the defined websites to utilize the git deploy system.

### 4. Execute ansible.

The playbook is setup with tags to make it easier to setup only what you need on a run.  For example if you've already run this at least once for a host, you probably don't need the init roles to execute again so you can simply do this:

```
ansible-playbook playbook.yml --skip-tags="init"
```

Or you may only want to setup the staging site first:

```
ansible-playbook playbook.yml --skip-tags="production"
```

Or you may have a new domain you want to setup on an existing server that was initialized by this playbook.  In which case you could just modify your `vars.yml` so `main_domain` has your new domain and execute:

```
ansible-playbook playbook.yml --tags="web"
```


The LetsEncrypt role requires your server to be publicly accessible with the given domains pointing to it. If you don't have this setup yet and are just getting the server setup for initial development you can skip letsencrypt setup by doing this:

```
ansible-playbook playbook.yml --skip-tags="ssl"
```

Then when you are ready to add ssl to existing websites after they are publicly available you can just setup the ssl configuration for those sites by doing:

```
ansible-playbook playbook.yml --tags="ssl"
```

Or you can optionally only set up ssl for staging via:

```
ansible-playbook playbook.yml --tags="ssl" --skip-tags="production"
```

Of course if _do_ have the server publicly accessible for your given domains and you just want to setup everything on a fresh server, then you can do just execute the full playbook:

```
ansible-playbook playbook.yml
```

So you can see with the tags, there's many options for how you use this playbook!

## Directory structure for created sites:

The following structure is used for the WordPress sites created:

```
.\home\webuser\www\
├── mysite.com (production)
    └── logs
        └── error.log
        └── access.log
    └── ...all other WordPress files/directories    
├── staging.mysite.com (staging)
    └── logs
        └── error.log
        └── access.log
    └── ...all other WordPress files/directories
├── letsencrypt
       
```

Note: the `letsencrypt` folder is not a WordPress site.  It is only accessible on requests to the `/.well-known/acme-challenge` route for any of your WordPress domains.  It's used by the letsencrypt service to authenticate your ssl certificate setup.

## Credits

Inspired by the playbooks, [WordPress Ansible](https://github.com/A5hleyRich/wordpress-ansible) and [WordPress Ansible Playbook](https://github.com/tlovett1/wordpress-ansible-playbook/tree/f4a5fd158b346aac830c72744a10b26807c53496)
