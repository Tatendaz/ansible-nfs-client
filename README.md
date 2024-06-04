Role Name: Ansible NFS client
=========

The role deploys an NFS client to host.

Requirements
------------

Ubuntu 22

Role Variables
--------------

Defaults/main.yml contains all variables you can set. You can set these variable to your own in your inventory or vars folder.


Example Playbook
----------------

Example of how to use the role :

    - hosts: all
      remote_user: root
      roles:
        - ansible-nfs-client


