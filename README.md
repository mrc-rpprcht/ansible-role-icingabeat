Ansible Role Icingabeat
=========

An easy to use Ansible Role to intsall and configure Icingabeat.

Requirements
------------

The role requires the current Icinga repositories. As the role is currently built to complement the Ansible Collection Icinga, a dependency on the Icinga repo role is built in. 
If the role is to be used without the collection, this dependency can simply be commented out or removed in meta/main.yml.

Role Variables
--------------

To apply the role, the variables in defaults/main.yml must be adapted to your own environment.

The variables to be adapted are

For Icingabeat:
icingabeat_host, icingabeat_port, icingabeat_user, icingabeat_password, icingabeat_ca_path

For Elasticsearch:
elasticsearch_hosts, elasticsearch_protocol, elasticsearch_ca_path, elasticsearch_username, elasticsearch_password

For Kibana:
kibana_host

Dependencies
------------

- icinga.icinga.repos

Example Playbook
----------------
Here is a working example playbook that rolls out the role together with the Ansible Collection icinga:

```yaml
- name: Installation of Icinga, Icinga Web and Icinga DB
  hosts: all
  become: yes
  # The "vars" section contains all variables required by the collection / roles
  vars:
      # Variables for MySQL (on Ubuntu 22.04 Server LTS) and icinga.repos (pre-tasks)
      mysql_innodb_file_format: barracuda
      mysql_innodb_large_prefix: 1
      mysql_innodb_file_per_table: 1
      mysql_packages:
        - mariadb-client
        - mariadb-server
        - python3-mysqldb
      mysql_users:
        - name: icingaweb
          host: "%"
          password: icingaweb
          priv: "icingaweb.*:ALL"
        - name: icingadb
          host: "%"
          password: icingadb
          priv: "icingadb.*:ALL"
      mysql_databases:
        - name: icingadb
        - name: icingaweb

      icinga_repo_stable: true
      icinga_repo_testing: false
      icinga_repo_snapshot: false

      # Variables for Icinga, Icinga DB and Icinga DB Redis (tasks)
      icinga2_confd: true
      icinga2_features:
        - name: icingadb
          host: 127.0.0.1
        - name: notification
        - name: checker
        - name: mainlog
        - name: api
          ca_host: none
          endpoints:
            - name: "{{ ansible_fqdn }}"
          zones:
            - name: "main"
              endpoints:
                - "{{ ansible_fqdn }}"
      icinga2_objects:
        icinga:
          - name: icingaweb
            type: ApiUser
            file: "conf.d/icingweb.conf"
            password: icingaweb
            permissions:
              - "actions/*"
              - "objects/query/*"
              - "objects/modify/*"
              - "status/query"
          - name: icingabeat
            type: ApiUser
            file: "conf.d/icingabeat.conf"
            password: icingabeat
            permissions:
              - "events/*"
              - "status/query"
      icingadb_database_type: mysql
      icingadb_database_host: localhost
      icingadb_database_user: icingadb
      icingadb_database_password: icingadb
      icingadb_database_import_schema: true
      icingadb_database_tls: false


      icingadb_redis_ca: 192.168.122.164
      icingadb_redis_host: 127.0.0.1
      icingadb_redis_port: 6380
  
      # Variables for Icinga Web (Post-Tasks)
      icingaweb2_db:
        type: mysql
        name: icingaweb
        host: 127.0.0.1
        user: icingaweb
        password: icingaweb
      icingaweb2_db_import_schema: true
      icingaweb2_admin_username: icingaweb
      icingaweb2_admin_password: icingaweb
      icingaweb2_modules:
        icingadb:
          enabled: true
          source: package
          commandtransports:
            icinga:
              transport: api
              host: 127.0.0.1
              username: icingaweb
              password: icingaweb
          config:
            icingadb:
              resource: icingadb
            redis:
              tls: '0'
          redis:
            redis1:
              host: "127.0.0.1"
      icingaweb2_resources:
        icingadb:
          type: db
          db: mysql
          host: localhost
          dbname: icingadb
          username: icingadb
          password: icingadb
          use_ssl: "0"
          charset: utf8
   
            
      icinga_monitoring_plugins_check_commands:
            - disk
            - dns
            - tcp
            - uptime
            - ping

      elasticsearch_password: "SECURE_PASSWORD_HERE
                 
  # Determining the "correct" sequence of operations
  # Correct refers to the technical dependencies during the installation of Icinga  pre_tasks:
    - name: Entry in /etc/hosts
      ansible.builtin.lineinfile:
        path: /etc/hosts
        search_string: "elastic"
        line: "192.168.122.224 elastic"
        owner: root
        group: root
        # mode: "0644"
  
    - name: Prepare the required Icinga repos
      ansible.builtin.include_role:
        name: icinga.icinga.repos

    - name: Installation and configuration of MySQL
      ansible.builtin.include_role:
        name: geerlingguy.mysql

  tasks:
    - name: Install and configure Icinga
      ansible.builtin.include_role:
        name: icinga.icinga.icinga2

    - name: Setting up Icinga DB
      ansible.builtin.include_role:
        name: icinga.icinga.icingadb

    - name: Installation and configuration of Icinga DB Redis
      ansible.builtin.include_role:
        name: icinga.icinga.icingadb_redis

    - name: Installation and configuration of Icingabeat
      ansible.builtin.include_role:
        name: icinga.icinga.icingabeat

  post_tasks:
    - name: Set up Icinga Web
      ansible.builtin.include_role:
        name: icinga.icinga.icingaweb2

    - name: Install monitoring plugins
      ansible.builtin.include_role:
        name: icinga.icinga.monitoring_plugins
```

License
-------

Apache-2.0

Author Information
------------------

Name: Marc Rupprecht
Company: NETWAYS Professional Services GmbH
