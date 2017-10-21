# WordPress Ansible Playbook 

This repository contains an ansible playbook for provisioning a WordPress based server with the following:

- User Setup (used for all web related server activity)
- SSH Hardening
- Nginx with HTTP/2 and solid default configs.
- PHP 7.1
- MariaDB
- WP-CLI
- Git
- Lets Encrypt for SSL certs (and a cron for automatic renewal)
- A production and staging site with the given domains.

### Credits

Inspired by the playbooks, [WordPress Ansible](https://github.com/A5hleyRich/wordpress-ansible) and [WordPress Ansible Playbook](https://github.com/tlovett1/wordpress-ansible-playbook/tree/f4a5fd158b346aac830c72744a10b26807c53496)
