+++
title =  "cronjobs"
+++

Cronjobs are usefull for different maintenance tasks. This page lists some tipps and tricks about them.

## Scheduling cronjobs

When you have seen the cron scheduling syntax `*/1 * * * *` you know how confusing it can be to find the correct schedule. Tipp: use [https://crontab.guru/](https://crontab.guru/) to figure out your schedule.

## Send mails from cronjobs

We are using postfix for this. ssmtp would be more minimal but I have tried it extensively and it didn't work.

Used [this](https://devanswers.co/postfix-external-smtp-server/) tutorial and [this](https://devanswers.co/postfix-statusbounced-unknown-user-user/).

Some notes:

- My mail account is root@technat.ch
- My mail provider is [Infomanaik](https://www.infomaniak.com/en) using `mail.infomaniak.com:465` as SMTP server
- My mail provider requires TLS, STARTTLS and the FROM address field must be root@technat.cloud or any alias defined in their UI.

Start by installing programms:

```bash
sudo apt-get install mailutils postfix -y
```

Choose "Internet Site" and "localhost".

Then add the following content into `/etc/postfix/sasl_passwd`:

```bash
[mail.infomaniak.com]:465 username:password
```

Save it and run the following command:

```bash
sudo postmap /etc/postfix/sasl_passwd
```

This hashes the credentials and creates the file `/etc/postfix/sasl_passwd.db`.

Anyway because we work with passwords let's restrict the permissions to those files:

```bash
sudo chown root:root /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
sudo chmod 0600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
```

And then adjust the following config directives in `/etc/postfix/main.cf`:

```bash
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
smtp_use_tls = yes
smtp_tls_wrappermode = yes
smtp_tls_security_level = encrypt
relayhost = [mail.infomaniak.com]:465
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
myhostname = localhost
myorigin = /etc/mailname
```

Also add `technat.ch` (the domain you are sending mails from) in `/etc/mailname`.

And then restart the service: `sudo systemctl restart postfix`.

Finally edit the `/etc/aliases`:

```bash
postmaster: root
technat: root
www-data: root
root: root@technat.cloud
```

And apply that using `sudo newaliases`.

Note: If you send mails from cronjobs, they will come from username@technat.cloud where username is the user that runs the cronjob. This means that if you use a mail provider that checks if the from address matches the mail of the account you must either add an alias or find out how to replace the from address.
