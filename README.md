# Ansible role to set up a mail server
This [ansible](https://www.ansible.com/) role installs a mail server as described
[in this blog post by Thomas Leistner](https://thomas-leister.de/mailserver-unter-ubuntu-16.04/).

## SSL
SSL certificates and keys are expected in `/etc/myssl/$FQDN.crt` and `/etc/myssl/$FQDN.crt` on the mail host
(where $FQDN is the hosts FQDN). This can be change by setting the `{{ ssl_directory }}`, `{{ ssl_directory }}` or
`{{ ssl_directory }}` variable to a different path.
If these files do not exist, a self-signed certificate will be created for initial use,
these are not recommended for a production setup. If you require a certificate signed by a trusted CA, try
[Let's Encrypt](https://letsencrypt.org/).

## Variables
The following variables have to be set by you in order to use this role:

| Variable | Explanation |
| ------------- | ------------- |
| dbserver_root_pw | password for the root user of the database |
| mailserver_sql_vmail_password | password for the vmail user in the database |
| milter_sql_spamass_password | password for the spamassassin user in the database |
| mailserver_hostname | hostname of the mail server (e.g. `mail`) |
| mailserver_domain | domain of the mail server (e.g. `example.com`) |

A lot more variables are available to customize the mail server installation, you can find them in the `defaults/main.yml` of this role
and its dependencies.

## Passwords
Instead of saving passwords in plain-text in `vars/vars.yml`, consider using the
[ansible vault](https://docs.ansible.com/ansible/playbooks_vault.html). 

For a quick-start to the vault, you can execute `ansible-vault create vars/vault.yml` and fill it like so:

```yml
mailserver_sql_vmail_password: foo
milter_sql_spamass_password: bar
dbserver_root_pw: baz
```
(Replace foo, bar and baz with secure passwords)

Now, whenever you run a playbook with this role, remember to use the `--ask-vault-pass` with `ansible-playbook`.

## Deployment
Before deployment, you **must** set passwords for the database users (see above).
Also this playbook assumes a default installation of Ubuntu Server 16.04 and hasn't been tested for any other distribution.

An example playbook could look like this:
```yml
---
- hosts: all
  become: yes
  roles:
    - ROCK5GmbH.mailserver
  vars:
    - vault.yml
```

To get this role and all its dependencies, you can use the ansible galaxy:

`ansible-galaxy install ROCK5GmbH.mailserver`

To deploy on a single host, it is sufficient to execute
```bash
ansible-playbook --ask-vault-pass -i $HOST, playbook.yml
```
where $HOST is the IP address or URL of the server.

If you want to deploy to multiple hosts, it is recommended to work with
[inventories](https://docs.ansible.com/ansible/intro_inventory.html), so you can set variables per host while keeping
common variables.

After the deployment, the mail server will be almost ready to go. All that's left now is to add actual domains and users to your SQL database.
First use `doveadm pw -s SHA512-CRYPT` to generate a hash for the password for the user.
To add the user, log in to your server then connect to the database via
```bash
mysql -u root -p
```
and enter the root password for the SQL database. Now add a domain (here: mysystems.tld) and a user (user1):
```sql
use vmail;
insert into domains (domain) values ('mysystems.tld');
insert into accounts (username, domain, password, quota, enabled, sendonly) values ('user1', 'mysystems.tld', '{SHA512-CRYPT}$kgid87hdenss', 2048, true, false);
```

## Further customization
This setup also allows for TLS policies in the SQL database, allowing you to set specific TLS policies for specific domains.
You can add a policy via
```sql
insert into tlspolicies (domain, policy, params) values ('gmx.de', 'secure', 'match=.gmx.net');
```
The different possible policies are listed and explained [here](http://www.postfix.org/TLS_README.html#client_tls_levels).
(The `match=.gmx.net` makes sure that postfix will check the certificate for `gmx.net`, since they do not have one for `gmx.de`)

You may also want to add your new mail server to your DNS (A(AAA) and MX record), as well as add an entry for [SPF](https://www.dynu.com/NetworkTools/SPFGenerator) and DKIM.
