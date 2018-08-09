# aspects_postfix

Install and configure postfix.

# Requirements

Set ```hash_behaviour=merge``` in your ansible.cfg file.

# Role Variables
## aspects_postfix_enabled
Run the tasks in the role.

Default is `False`.

Set to `True` to run the tasks.
## aspects_packages_packages
Dictionary/hash of packages to install or remove. Uses the aspects_packages role. 

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

### aspects_postfix_template_transport
Used to configured transport map files.

Set `aspects_postfix_template_transport` to `True` to enable tasks.

Defaults:

```yaml
aspects_postfix_template_transport:
  enabled: False
  templatefile: transport.j2
  templatepath: /etc/postfix/transport
```

Add the appropriate line to `main.cf` to let postfix know you are using transport maps.

For example:

```yaml
aspects_postfix_main_cf:
  00015transport_maps: |
    transport_maps = hash:/etc/postfix/transport
```

### aspects_postfix_transport
Dictionary/hash of transport map configuration blocks.

Use the following pattern:

```yaml
aspects_postfix_transport:
  <key>: |
    <configuration>
```

The dictionary is filtered through the jinja sort function. Use prefixes as appropriate to set the order the blocks apply in.

When the template task applies the configuration, the `postmap transport` handler will be fired. It runs `postmap {{ aspects_postfix_template_transport.templatepath }}` using the shell module.

# Example Playbook

```yaml
- hosts:
  - all
  vars:
    aspects_postfix_enabled: True
    aspects_postfix_authed_relay_enabled: True
    aspects_postfix_authed_relay: relay.example.org
    aspects_postfix_authed_relay_user: george_of_the_jungle
    aspects_postfix_authed_relay_password: watch_out_for_that_tree
    aspects_postfix_template_transport:
      enabled: True
    aspects_postfix_transport:
      123testing: |
        example.com      uucp:example
    aspects_postfix_main_cf:
      00000smtpd_banner: |
        smtpd_banner = $myhostname ESMTP $mail_name
      00001myhostname: |
        myhostname = {{ ansible_fqdn }}
      00002myorigin: |
        myorigin = $myhostname
      00003mydestination: |
        mydestination = {{ ansible_fqdn }}, localhost.localdomain, localhost
      00004mynetworks_style: |
        mynetworks_style = host
      00005boilerplate: |
        biff = no
        readme_directory = no
        # appending .domain is the MUA's job.
        append_dot_mydomain = no
      00006tlsparams: |
        # TLS parameters
        # Ubuntu 14.04, Ubuntu 16.04, and Debian 9 provide snakeoil certs and keys.
        # at /etc/ssl/certs/ssl-cert-snakeoil.pem and /etc/ssl/private/ssl-cert-snakeoil.key
        # if you wish to use them.
        smtpd_tls_cert_file=none
        # smtpd_tls_key_file=/path/to/key/file
        smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
        smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
        smtp_tls_security_level = encrypt
      00007aliases: |
        alias_maps = hash:/etc/aliases
        alias_database = hash:/etc/aliases
      00008mailmessagelimits: |
        mailbox_size_limit = 0
        message_size_limit = 40960000
      00009inetstuff: |
        inet_interfaces = loopback-only
        inet_protocols = all
      00010saslauth: |
        # Use sasl authentication.
        smtp_sasl_auth_enable = yes
        smtp_sasl_tls_security_options = noanonymous
        smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
      00011userelayhost: |
        relayhost = {{ aspects_postfix_authed_relay }}
      00015transport_map: |
        transport_maps = hash:{{ aspects_postfix_template_transport.templatepath }}
    aspects_postfix_master_cf:
#      00000header: |
#        # ==========================================================================
#        # service type  private unpriv  chroot  wakeup  maxproc command + args
#        #               (yes)   (yes)   (yes)   (never) (100)
#        # ==========================================================================
      00001smtp: |
        smtp      inet  n       -       -       -       -       smtpd
      00001smtpa: |
        # shows up after 00001smtp
      00001asmtp: |
        # shows up before 00001smtp
      00002pickup: |
        pickup    unix  n       -       -       60      1       pickup
      00003cleanup: |
        cleanup   unix  n       -       -       -       0       cleanup
      00004qmgr: |
        qmgr      unix  n       -       n       300     1       qmgr
      00005tlsmgr: |
        tlsmgr    unix  -       -       -       1000?   1       tlsmgr
      00006rewrite: |
        rewrite   unix  -       -       -       -       -       trivial-rewrite
      00007bounce: |
        bounce    unix  -       -       -       -       0       bounce
      00008defer: |
        defer     unix  -       -       -       -       0       bounce
      00009trace: |
        trace     unix  -       -       -       -       0       bounce
      00010verify: |
        verify    unix  -       -       -       -       1       verify
      00011flush: |
        flush     unix  n       -       -       1000?   0       flush
      00012proxymap: |
        proxymap  unix  -       -       n       -       -       proxymap
      00013proxywrite: |
        proxywrite unix -       -       n       -       1       proxymap
      00014smtp: |
        smtp      unix  -       -       -       -       -       smtp
      00015relay: |
        relay     unix  -       -       -       -       -       smtp
      00016showq: |
        showq     unix  n       -       -       -       -       showq
      00017error: |
        error     unix  -       -       -       -       -       error
      00018retry: |
        retry     unix  -       -       -       -       -       error
      00019discard: |
        discard   unix  -       -       -       -       -       discard
      00020local: |
        local     unix  -       n       n       -       -       local
      00021virtual: |
        virtual   unix  -       n       n       -       -       virtual
      00022lmtp: |
        lmtp      unix  -       -       -       -       -       lmtp
      00023anvil: |
        anvil     unix  -       -       -       -       1       anvil
      00024scache: |
        scache    unix  -       -       -       -       1       scache
  roles:
  - aspects_postfix
```

# License

MIT