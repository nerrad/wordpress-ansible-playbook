# WordPress Ansible Playbook 

This repository contains an ansible playbook for provisioning a WordPress based server with both a production and staging website, optional ssl certificates (provided free via letsencrypt), PHP 7.3, Mariadb, wp-cli, and nginx.

> Note: currently this is oriented towards a Ubuntu or Debian based box.

## Roles

The following roles are in this playbook.

| Role | Description | Tags |
|-------|------------|------|
| common | Installs latest for `fail2ban`, `git-core`, `letsencrypt`,  `unzip`, `python-pip`, `python-dev` | init
| user | Creates a web user and web user group and adds a provided .ssh key.  This user will be used for all web services (and web site folders) | init
| ssh |  Disables root login and password authentication for ssh (hardening) | init |
| nginx | Clones nginx config from the provided repository and configures default settings (including ssl global settings, catchall logs, and the default catchall config) | web, nginx |
|nginxsites | Configures the nginx conf files for the registered domains. While this role does setup both ssl and non-ssl configs for the provided domain(s), it only _activates_ the non-ssl config.  | nginx, web, sites      |
| php | Installs PHP 7.3 (by default) and all necessary php extensions WordPress needs.  Also modifies the php configs to set the web user etc. | init |
| mariadb | Installs `mariadb-server` and `python-mysqldb`, sets up root password for all root accounts (generated server side), creates .my.cnf with root password creds (to `/root/.my.cnf` on the server), deletes anonymous mysql server user for server hostname and for local host and removes the mysql test database. | init |
| wp-cli | Installs wp-cli | init
| wordpress | Installs and sets up wp sites for given domains.  You will end up with working vanilla WordPress sites for those domains on the server | web, wordpress |
| letsencrypt | This will create, authorize, retrieve and setup ssl certificate for the given domain(s) using the letsencrypt service.  It will deactivate the nonssl nginx configuration for the domain and activate the ssl configuration.  It will also setup a cron job for automatically renewing the certificate(s). | web, ssl |
| importdb | This will import a mysql database export into designated site dataabases as defined in the `wp_db_import` variable found in your `vars.yml` file (see variables section below for more info) | web, wordpress, import
| gitdeploy | Set's up a git deploy system on the server for pushing changes to your site(s) via git. Utilizes the `gitdeploy` variable defined in your `vars.yml` file with domains registered to use it. [Read more details here.](roles/gitdeploy/README.md) | deploy 
| logentries | This will setup the log entries service and follow any logs that you've already defined. | logentries
| geerlingguy.node | adds nodejs to the server | node
| grunt | adds grunt-cli to the server. | grunt
| geerlingguy.composer | adds composer to the server | composer
| geerlingguy.redis | adds redis to the server |  redis
| php-redis | adds the php7.3 module for redis to the server | redis
| buildmachine.webhook.listener | Sets up the [webhook listener](https://github.com/nerrad/buildmachine-webhook-listener) for the Grunt Buildmachine for WordPress plugins | webhook, buildmachine
| grunt-buildmachine | Sets up the [Grunt Buildmachine for WordPress Plugins](https://github.com/eventespresso/grunt-wp-plugin-buildmachine) service. | buildmachine, gruntbm

## Usage

To get started, you just need to do the following:

### 1. Create a server that has a sudo user already set up.

Make sure that whatever machine you are connecting from as the ansible host has its public key setup on any hosts you are deploying to.

It goes without saying that this is an ansible playbook so its assumed there's already some familiarity with how that works.

### 2. Copy `hosts.example` to `hosts` and add your server info. 

### 3. Copy `vars-sample.yml` to `vars.yml` and add your own values for the variables.

> **Note:** Currently the default `nginx_config_repo` will _not_ work as is until [submitted](https://github.com/A5hleyRich/wordpress-nginx/pull/14) [pull requests](https://github.com/A5hleyRich/wordpress-nginx/pull/13) for that repository are merged in. In the meantime, you can use https://github.com/nerrad/wordpress-nginx.git and the `test-merge-le-phpup` branch there.

These variables are used throughout the playbook.  The vars-sample.yml file is documented inline so you can read more about the various variables there.  Take note, that the `domains` variable is a dictionary array where each domain is its own entity.  So you can add multiple domains as a part of your configuration and this playbook will automatically loop through them for the various roles in use.

**Note:** the roles imported from elsewhere have variables that can be set as well.  You can find more info about those variables within the `README.md` found in that roles directory.  This includes the following roles:

- geerlingguy.redis
- geerlingguy.composer
- geerlingguy.nodejs
- logentries

### 4. Execute ansible.

The playbook is setup with tags to make it easier to setup only what you need on a run.  For example if you've already run this at least once for a host, you probably don't need the init roles to execute again so you can simply do this:

```
ansible-playbook playbook.yml --skip-tags="init"
```

Or you may have a new domain you want to setup on an existing server that was initialized by this playbook.  In which case you could just modify your `vars.yml` so there is a new domain entity in the `domains` array and then run:

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

Of course if _do_ have the server publicly accessible for your given domains and you just want to setup everything on a fresh server, then you can do just execute the full playbook:

```
ansible-playbook playbook.yml
```

You can also update or downgrade the PHP version by setting the `php_fpm.version` variable in your `vars.yml` and running just the `init` tag:
```
ansible-playbook playbook.yml --tags="init"
```

So you can see with the tags, there's many options for how you use this playbook!

### Running play for only specific domains
It is also possible to only include or exclude specific domains defined in your `vars.yml` file.  One scenario where this would be useful is if you wanted to speed up the deploy of a new site for a domain to the server.  

To do this, you utilize the `-e` flag on the `ansible-playbook` command to pass along extra variables that this playbook will listen for:

`include_domains`: Any domains listed here will be _included_ in the play.
`exclude_domains`: Any domains listed here will be _excluded_ from the play.  Note: if you have the same domain listed with both variables, exclude will override include.

**Example A: Including domains**

In the following example, only the `testc.mydomain.com` and `testd.mydomain.com` sites from your `vars.yml` file will have the play executed on them.

```
ansible-playbook playbook.yml -e "include_domains=testc.mydomain.com,testd.mydomain.com"
```

**Example B: Excluding domains**

In the following example, both `testc.mydomain.com` and `testd.mydomain.com` sites from your `vars.yml` file will be excluded form the playbook execution.

```
ansible-playbook playbook.yml -e "exclude_domains=testc.mydomain.com,testd.mydomain.com"
```


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

Other Roles used:

- logentries role: https://github.com/ricbra/ansible-logentries
- geerlingguy.nodejs: https://github.com/geerlingguy/ansible-role-nodejs
- geerlingguy.composer: https://github.com/geerlingguy/ansible-role-composer
- geerlingguy.redis: https://github.com/geerlingguy/ansible-role-redis
