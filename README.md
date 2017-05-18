# Ansible Playbook for Mailserver
This [ansible](https://www.ansible.com/) playbook installs a mail server as described
[in this blog post by Thomas Leistner](https://thomas-leister.de/mailserver-unter-ubuntu-16.04/).

## SSL
SSL certificates and keys are expected in `/etc/ssl/$FQDN.crt` and `/etc/ssl/$FQDN.crt` on the mail host
(where $FQDN is the hosts FQDN).
If these files do not exist, a self-signed certificate will be created for initial use,
these are not recommended for a production setup. If you require a certificate signed by a trusted CA, try
[Let's Encrypt](https://letsencrypt.org/).

## Variables
To customize the mail server with details specific to your setup (your domain, etc.), please consult the `vars/vars.yml` file.

## Passwords
Instead of saving passwords in plain-text in `vars/vars.yml`, consider using the
[ansible vault](https://docs.ansible.com/ansible/playbooks_vault.html). 

For a quick-start to the vault, you can execute `ansible-vault create vars/vault.yml` and fill it like so:

```yml
sql_dovecot_user_password: foo
sql_postfix_user_password: bar
sql_root_pass: baz
```
(Replace foo, bar and baz with secure passwords)

Now, whenever you run this playbook, remember to use the `--ask-vault-pass` with `ansible-playbook`.

If you insist on storing passwords in plain text, you must uncomment the respecitve variables in `vars/vars.yml` and remove the include for `vars/vault.yml` from the playbook.

## Deployment
Before deployment, you *must* set passwords for the database users (see above).
Also this playbook assumes a default installation of Ubuntu Server 16.04 and hasn't been tested for any other distribution.

To deploy on a single host, it is sufficient to execute
```bash
ansible-playbook --ask-vault-pass -i $HOST, playbook.yml
```
where $HOST is the IP address or URL of the server.

If you want to deploy to multiple hosts, it might be advisable to work with
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

You may also want to add your new mailserver to your DNS (A(AAA) and MX record), as well as add an entry for [SPF](https://www.dynu.com/NetworkTools/SPFGenerator) and DKIM.
