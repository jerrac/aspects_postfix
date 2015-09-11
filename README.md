aspects_postfix
=========

Install and configure postfix. Templates master.cf, main.cf, and any other configuration files you add. (Like for mysql auth.)

Requirements
------------

Set ```hash_behaviour=merge``` in your ansible.cfg file.

```aspects_postfix_enabled``` must be set to ```True``` for the tasks to run.

Role Variables
--------------

See the ```defaults/main.yml`` file for available variables. 

Dependencies
------------

* [aspects_iptables](https://github.com/LaneCommunityCollege/aspects_iptables) is a dependency when using it to manage iptables.

Example Playbook
----------------


    - hosts: servers
      roles:
         - { role: aspects_postfix, aspects_postfix_enabled: True }
      vars:
        aspects_postfix_main_cf:
        mydestination: |
          mydestination = {{ ansible_fqdn }}, localhost.example.tld, localhost
        aspects_postfix_master_cf:
          nosmtpchroot: |
              # ==========================================================================
              # service type  private unpriv  chroot  wakeup  maxproc command + args
              #               (yes)   (yes)   (yes)   (never) (100)
              # ==========================================================================
              smtp      inet  n       -       n       -       -       smtpd

License
-------

MIT
