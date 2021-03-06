# ratelimit-policyd

A Sender rate limit policy daemon for Postfix.

Copyright (c) Ayoola Falola (http://www.ayoo.la)

## Credits

This project was forked originally from  [bejelith/send_rate_policyd](https://github.com/bejelith/send_rate_policyd). All credits go to [Simone Caruso](http://www.simonecaruso.com). 

More features added by [onlime/ratelimit-policyd](https://github.com/onlime/ratelimit-policyd). Credits go to [Onlime Webhosting](http://www.onlime.ch)

## Purpose

This small Perl daemon **limits the number of emails** sent by users through your Postfix server, and store message quota in a RDMS system (MySQL). It counts the number of recipients for each sent email. You can setup a send rate per user or sender domain (via SASL username) on **daily/weekly/monthly** basis.

**The program uses the Postfix policy delegation protocol to control access to the mail system before a message has been accepted (please visit [SMTPD_POLICY_README.html](http://www.postfix.org/SMTPD_POLICY_README.html) for more information).**

For a long time we were using Postfix-Policyd v1 (the old 1.82) in production instead, but that project was no longer maintained and the successor [PolicyD v2 (codename "cluebringer")](http://wiki.policyd.org/) got overly complex and badly documented. Also, PolicyD seems to have been abandoned since 2013.

ratelimit-policyd will never be as feature-rich as other policy daemons. Its main purpose is to limit the number of emails per account, nothing more and nothing less. We focus on performance and simplicity.

**This daemon caches the quota in memory, so you don't need to worry about I/O operations!**

## New Features by [onlime/ratelimit-policyd](https://github.com/onlime/ratelimit-policyd) 

The original forked code from [bejelith/send_rate_policyd](https://github.com/bejelith/send_rate_policyd) was improved with the following new features:

- automatically inserts new SASL-users (upon first email sent)
- Debian default init.d startscript
- added installer and documentation
- bugfix: weekly mode did not work (expiry date was not correctly calculated)
- bugfix: counters did not get reset after expiry
- additional information in DB: updated timestamp
- added view_ratelimit in DB to make Unix timestamps human readable (default datetime format)
- syslog messaging (similar to Postfix-policyd) including all relevant information and counter/quota
- more detailed logging
- added logrotation script for /var/log/ratelimit-policyd.log
- added flag in ratelimit DB table to make specific quotas persistent (all others will get reset to default after expiry)
- continue raising counter even in over quota state

## New Features by [ayoolafalola/ratelimit-policyd](https://github.com/ayoolafalola/ratelimit-policyd) 

- Sendmail to admin on overusage
- Sendmail to overusage user
- Allows to set quota for email and domain at the same time
- Separate domain default quota from email default quota
- Separate config file for seamless upgrade
- Updated Installation doc to include plugin installations


## Installation

Recommended installation:


Install required plugins

```bash
sudo apt-get -y install mysql-server insserv libswitch-perl libdbd-mysql-perl
```

Create the DB schema and user:

```bash
mysql -u root -p < mysql-schema.sql
```
Run this SQL to create database user. Change '************' with your chosen password

```sql
GRANT USAGE ON *.* TO policyd@'localhost' IDENTIFIED BY '************';
GRANT SELECT, INSERT, UPDATE, DELETE ON policyd.* TO policyd@'localhost';
```

```bash

# download ratelimit-policyd 
cd /opt/
cp ratelimit-policyd/config.pl ratelimit-policy-config.pl
rm -rf ratelimit-policyd
git clone https://github.com/ayoolafalola/ratelimit-policyd.git ratelimit-policyd
cp ratelimit-policy-config.pl ratelimit-policyd/config.pl
cd ratelimit-policyd
chmod +x install.sh

# Install
./install.sh

```


Adjust configuration options in ```config.pl```:
Change '************' with your chosen password

```perl
#!/usr/bin/perl

(
    'listen_address' => '127.0.0.1',
    'port' => '10032',
    'db_user' => 'policyd',
    'db_passwd' => '************',
    's_key_type' => 'all',
    'deltaconf' => 'daily',
    'defaultquota' => 100,
    'defaultDomainQuota' => '200',
    'sendMailNotificationFrom' => 'mail@localhost',
    'sendMailNotificationTo' => 'root@localhost',
);
```

**Take care of using a port higher than 1024 to run the script as non-root (our init script runs it as user "postfix").**

In most cases, the default configuration should be fine. Just don't forget to paste your DB password in ``$db_password``.

Now, start the daemon:

```bash
service ratelimit-policyd start
```

## Testing

Check if the daemon is really running:

```bash
netstat -tl | grep 10032
```
Above command should output something like the following:
tcp        0      0 localhost.localdo:10032 *:*                     LISTEN


```bash
cat /var/run/ratelimit-policyd.pid
```
Above command should output something like the following:
30566

```bash
ps aux | grep daemon.pl
```
Above command should output something like the following:

postfix  30566  0.4  0.1 176264 19304 ?        Ssl  14:37   0:00 /opt/send_rate_policyd/daemon.pl

```bash
pstree -p | grep ratelimit
```

Above command should output something like the following:

```
init(1)-+-/opt/ratelimit-(11298)-+-{/opt/ratelimit-}(11300)
        |                        |-{/opt/ratelimit-}(11301)
        |                        |-{/opt/ratelimit-}(11302)
        |                        |-{/opt/ratelimit-}(14834)
        |                        |-{/opt/ratelimit-}(15001)
        |                        |-{/opt/ratelimit-}(15027)
        |                        |-{/opt/ratelimit-}(15058)
        |                        `-{/opt/ratelimit-}(15065)
```
Print the cache content (in shared memory) with update statistics:

```bash
service ratelimit-policyd status
```
Above command should output something like the following:

Printing shm:
Domain		:	Quota	:	Used	:	Expire
Threads running: 6, Threads waiting: 2


## Postfix Configuration

Modify the postfix data restriction class ```smtpd_data_restrictions``` like the following, ```/etc/postfix/main.cf```:

```
smtpd_data_restrictions = check_policy_service inet:127.0.0.1:10032
```

sample configuration (using ratelimitpolicyd as alias as smtpd_data_restrictions does not allow any whitespace):

```
smtpd_restriction_classes = ratelimitpolicyd
ratelimitpolicyd = check_policy_service inet:127.0.0.1:10032

smtpd_data_restrictions =
        reject_unauth_pipelining,
        ratelimitpolicyd,
        permit
```

If you're sure that ratelimit-policyd is really running, restart Postfix:

```
service postfix restart
```

## Logging

Detailed logging is written to ``/var/log/ratelimit-policyd.log```. In addition, the most important information including the counter status is written to syslog:

Check ratelimit-policyd log

```
tail -f /var/log/ratelimit-policyd.log 
```
Above command should output something like the following:

Sat Jan 10 12:08:37 2015 Looking for demo@example.com
Sat Jan 10 12:08:37 2015 07F452AC009F: client=4-3.2-1.cust.example.com[1.2.3.4], sasl_method=PLAIN, sasl_username=demo@example.com, recipient_count=1, curr_count=6/1000, status=UPDATE

Check systemlog

```
tail -f /var/log/syslog | grep ratelimit-policyd 
```

Above command should output something like the following:

Jan 10 12:08:37 mx1 ratelimit-policyd[2552]: 07F452AC009F: client=4-3.2-1.cust.example.com[1.2.3.4], sasl_method=PLAIN, sasl_username=demo@example.com, recipient_count=1, curr_count=6/1000, status=UPDATE

