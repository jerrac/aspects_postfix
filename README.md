# aspects_postfix

Install and configure postfix.

# Requirements

Set ```hash_behaviour=merge``` in your ansible.cfg file.

# Role Variables
## aspects_postfix_enabled
Run the tasks in the role.

Default is `False`.

Set to `True` to run the tasks.
## aspects_postfix_packages
Dictionary/hash of packages to install or remove. Use the following pattern to add your own packages to the list. 
```yaml
aspects_firewalld_packages:
  <package key>:
    state: "<present or latest>"
    <ansible_distribution>:
      <ansible_distribution_version or ansible_distribution_major_version>: "<package name>"
```
## Configuration Block Variables
Each is a dictionary/hash that contain blocks of postfix configuration. Use the following pattern if you wish to add your own templates to the list.
```yaml
<variable>:
  <key>: |
    # configuration goes here.
```
| variable name | contains | [defaults/main.yml](defaults/main.yml) value |
| ------------- | -------- | -------------------------------------------- |
| aspects_postfix_master_cf | configuration for the master.cf file. | `{}` |
| aspects_postfix_main_cf | configuration for the main.cf file. | `{}` |
| aspects_postfix_mysql_virtual_alias_maps_conf | If you are using the mysql backend, this is where you put the mysql query for aliases. | `{}` |
| aspects_postfix_mysql_virtual_domains_maps_conf | If you are using the mysql backend, this is where you put the mysql query for domains. | `{}` |
| aspects_postfix_mysql_virtual_mailbox_maps_conf | If you are using the mysql backend, this is where you put the mysql query for mailboxes. | `{}` |
| aspects_postfix_mysql_virtual_transport_maps_conf | If you are using the mysql backend, this is where you put the mysql query for transports. | `{}` |

> **Important:** Templates in this role pass the dictionary/hashes through the jinja sort filter. So you can control the order blocks are written by prefixing them appropriately.

## aspects_postfix_templates
A dictionary/hash of templates and destinations. Default values are set in [defaults/main.yml](defaults/main.yml).

Use the following pattern to add your own.

```yaml
aspects_postfix_templates:
  <key>
    enabled: <True|False>
    templatefile: <The jinja template file to use>
    templatepath: </path/to/dest/file>
``` 

| key | enabled | templatefile | templatepath |
| --- | ------- | ------------ | ------------ |
| master_cf | True | master.cf.j2 | /etc/postfix/master.cf |
| main_cf | True | main.cf.j2 | /etc/postfix/main.cf |
| mysql_virtual_alias_maps_conf | False | mysql_virtual_alias_maps.conf.j2 | /etc/postfix/mysql_virtual_alias_maps.conf |
| mysql_virtual_domains_maps_conf | False | mysql_virtual_domains_maps.conf.j2 | /etc/postfix/mysql_virtual_domains_maps.conf |
| mysql_virtual_mailbox_maps_conf | False | mysql_virtual_mailbox_maps.conf.j2 | /etc/postfix/mysql_virtual_mailbox_maps.conf |
| mysql_virtual_transport_maps_conf | False | mysql_virtual_transport_maps.conf.j2 | /etc/postfix/mysql_virtual_transport_maps.conf |

## Authenticated Relay Configuration
Postfix can authenticate to a remote relay server when sending email. Use the following to configure that functionality.
### aspects_postfix_authed_relay_enabled
Run the authenticated relay tasks, or not.

Default: `False`

Set to `True` if you want the tasks to run.

### aspects_postfix_authed_relay
String. The hostname of the relay server you are authenticating to.
### aspects_postfix_authed_relay_user
String. The username of the user you are authenticating with.
### aspects_postfix_authed_relay_password
String. The password of the user you are authenticating with. 
> I suggest placing this value in a separate file encrypted by ansible-vault.

# Example Playbook

```yaml
- hosts:
  - all
  vars:
    aspects_postfix_enabled: True
    aspects_postfix_main_cf:
      alldefault: |
        myhostname = {{ ansible_fqdn }}
        myorigin = $myhostname
        smtpd_banner = $myhostname ESMTP $mail_name
        biff = no
        # appending .domain is the MUA's job.
        append_dot_mydomain = no
        readme_directory = no
        # TLS parameters
        # Ubuntu 14.04, Ubuntu 16.04, and Debian 9 provide snakeoil certs and keys.
        # at /etc/ssl/certs/ssl-cert-snakeoil.pem and /etc/ssl/private/ssl-cert-snakeoil.key
        # if you wish to use them.
        smtpd_tls_cert_file=none
        # smtpd_tls_key_file=/path/to/key/file
        smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
        smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
        # See /usr/share/doc/postfix/TLS_README.gz in the postfix-doc package for
        # information on enabling SSL in the smtp client.
        alias_maps = hash:/etc/aliases
        alias_database = hash:/etc/aliases
        mydestination = {{ ansible_fqdn }}, localhost.localdomain, localhost
        relayhost = mailrelay.example.org
        # Use sasl authentication.
        smtp_sasl_auth_enable = yes
        smtp_tls_security_level = encrypt
        smtp_sasl_tls_security_options = noanonymous
        smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
        mynetworks_style = host
        mailbox_size_limit = 0
        message_size_limit = 40960000
        inet_interfaces = loopback-only
        inet_protocols = all
    aspects_postfix_master_cf:
      alldefault: |
        # Postfix master process configuration file.  For details on the format
        # of the file, see the master(5) manual page (command: "man 5 master" or
        # on-line: http://www.postfix.org/master.5.html).
        #
        # Do not forget to execute "postfix reload" after editing this file.
        #
        # ==========================================================================
        # service type  private unpriv  chroot  wakeup  maxproc command + args
        #               (yes)   (yes)   (yes)   (never) (100)
        # ==========================================================================
        smtp      inet  n       -       -       -       -       smtpd
        #smtp      inet  n       -       -       -       1       postscreen
        #smtpd     pass  -       -       -       -       -       smtpd
        #dnsblog   unix  -       -       -       -       0       dnsblog
        #tlsproxy  unix  -       -       -       -       0       tlsproxy
        #submission inet n       -       -       -       -       smtpd
        #  -o syslog_name=postfix/submission
        #  -o smtpd_tls_security_level=encrypt
        #  -o smtpd_sasl_auth_enable=yes
        #  -o smtpd_reject_unlisted_recipient=no
        #  -o smtpd_client_restrictions=$mua_client_restrictions
        #  -o smtpd_helo_restrictions=$mua_helo_restrictions
        #  -o smtpd_sender_restrictions=$mua_sender_restrictions
        #  -o smtpd_recipient_restrictions=
        #  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
        #  -o milter_macro_daemon_name=ORIGINATING
        #smtps     inet  n       -       -       -       -       smtpd
        #  -o syslog_name=postfix/smtps
        #  -o smtpd_tls_wrappermode=yes
        #  -o smtpd_sasl_auth_enable=yes
        #  -o smtpd_reject_unlisted_recipient=no
        #  -o smtpd_client_restrictions=$mua_client_restrictions
        #  -o smtpd_helo_restrictions=$mua_helo_restrictions
        #  -o smtpd_sender_restrictions=$mua_sender_restrictions
        #  -o smtpd_recipient_restrictions=
        #  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
        #  -o milter_macro_daemon_name=ORIGINATING
        #628       inet  n       -       -       -       -       qmqpd
        pickup    unix  n       -       -       60      1       pickup
        cleanup   unix  n       -       -       -       0       cleanup
        qmgr      unix  n       -       n       300     1       qmgr
        #qmgr     unix  n       -       n       300     1       oqmgr
        tlsmgr    unix  -       -       -       1000?   1       tlsmgr
        rewrite   unix  -       -       -       -       -       trivial-rewrite
        bounce    unix  -       -       -       -       0       bounce
        defer     unix  -       -       -       -       0       bounce
        trace     unix  -       -       -       -       0       bounce
        verify    unix  -       -       -       -       1       verify
        flush     unix  n       -       -       1000?   0       flush
        proxymap  unix  -       -       n       -       -       proxymap
        proxywrite unix -       -       n       -       1       proxymap
        smtp      unix  -       -       -       -       -       smtp
        relay     unix  -       -       -       -       -       smtp
        #       -o smtp_helo_timeout=5 -o smtp_connect_timeout=5
        showq     unix  n       -       -       -       -       showq
        error     unix  -       -       -       -       -       error
        retry     unix  -       -       -       -       -       error
        discard   unix  -       -       -       -       -       discard
        local     unix  -       n       n       -       -       local
        virtual   unix  -       n       n       -       -       virtual
        lmtp      unix  -       -       -       -       -       lmtp
        anvil     unix  -       -       -       -       1       anvil
        scache    unix  -       -       -       -       1       scache
        #
        # ====================================================================
        # Interfaces to non-Postfix software. Be sure to examine the manual
        # pages of the non-Postfix software to find out what options it wants.
        #
        # Many of the following services use the Postfix pipe(8) delivery
        # agent.  See the pipe(8) man page for information about ${recipient}
        # and other message envelope options.
        # ====================================================================
        #
        # maildrop. See the Postfix MAILDROP_README file for details.
        # Also specify in main.cf: maildrop_destination_recipient_limit=1
        #
        maildrop  unix  -       n       n       -       -       pipe
          flags=DRhu user=vmail argv=/usr/bin/maildrop -d ${recipient}
        #
        # ====================================================================
        #
        # Recent Cyrus versions can use the existing "lmtp" master.cf entry.
        #
        # Specify in cyrus.conf:
        #   lmtp    cmd="lmtpd -a" listen="localhost:lmtp" proto=tcp4
        #
        # Specify in main.cf one or more of the following:
        #  mailbox_transport = lmtp:inet:localhost
        #  virtual_transport = lmtp:inet:localhost
        #
        # ====================================================================
        #
        # Cyrus 2.1.5 (Amos Gouaux)
        # Also specify in main.cf: cyrus_destination_recipient_limit=1
        #
        #cyrus     unix  -       n       n       -       -       pipe
        #  user=cyrus argv=/cyrus/bin/deliver -e -r ${sender} -m ${extension} ${user}
        #
        # ====================================================================
        # Old example of delivery via Cyrus.
        #
        #old-cyrus unix  -       n       n       -       -       pipe
        #  flags=R user=cyrus argv=/cyrus/bin/deliver -e -m ${extension} ${user}
        #
        # ====================================================================
        #
        # See the Postfix UUCP_README file for configuration details.
        #
        uucp      unix  -       n       n       -       -       pipe
          flags=Fqhu user=uucp argv=uux -r -n -z -a$sender - $nexthop!rmail ($recipient)
        #
        # Other external delivery methods.
        #
        ifmail    unix  -       n       n       -       -       pipe
          flags=F user=ftn argv=/usr/lib/ifmail/ifmail -r $nexthop ($recipient)
        bsmtp     unix  -       n       n       -       -       pipe
          flags=Fq. user=bsmtp argv=/usr/lib/bsmtp/bsmtp -t$nexthop -f$sender $recipient
        scalemail-backend unix	-	n	n	-	2	pipe
          flags=R user=scalemail argv=/usr/lib/scalemail/bin/scalemail-store ${nexthop} ${user} ${extension}
        mailman   unix  -       n       n       -       -       pipe
          flags=FR user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py
          ${nexthop} ${user}
  roles:
  - aspects_postfix
```

# License

MIT