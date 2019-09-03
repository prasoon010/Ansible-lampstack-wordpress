# Ansible-lampstack-wordpress using tags
This Ansible playbook will install lampstack and install a wordpress site in Amazon linux EC2. Using tags 'lamp' and 'lamp-wordpress' we have better control over installation. The tag 'lamp' the playbook will install lampstack alone. 

# Variables:
----------
```
#Mysql variables:
mysql_root: mysqlroot123

#Wordpress variables
wordpress_database: wordpress
wordpress_user: wpuser
wordpress_password: wpuser324

#Apache variables
domain: www.example.com
domain_owner: apache
domain_group: apache
```

Playbook:
---------
```

---
- name: "Playbook for lampstack-wordpress installation with removal"
  hosts: amazon
  become: yes
  vars:
    domain: www.example.com
    mysql_root: mysqlroot123
    wordpress_database: wordpress
    wordpress_user: wpuser
    wordpress_password: wpuser324
    domain_owner: apache
    domain_group: apache

  tasks:
    - name: "Lampstack - software package installation"
      yum:
        name:
          - httpd
          - php
          - php-mysql
          - MySQL-python
          - mariadb-server
        state: present
      tags:
        - lamp
        - lamp-wordpress

    - name: "Lampstack - Apache virtualhost creation"
      template:
        src: virtualhost.j2
        dest: "/etc/httpd/conf.d/{{domain}}.conf"
      tags:
        - lamp
        - lamp-wordpress

    - name: "Lampstack - Document root creation"
      file:
        path: "/var/www/html/{{domain}}"
        state: directory
        owner: apache
        group: apache
      tags:
        - lamp
        - lamp-wordpress

    - name: "Lampstack - creating info.php file"
      copy:
        content: '<?php phpinfo(); ?>'
        dest: "/var/www/html/{{domain}}/info.php"
        owner: apache
        group: apache
      tags:
        - lamp
        - lamp-wordpress

    - name: "Lampstack - restarting/enabling services"
      service:
        name: "{{ item }}"
        state: restarted
        enabled: yes
      with_items:
          - httpd
          - mariadb
      tags:
        - lamp
        - lamp-wordpress


    - name: "Lampstack - reseting mariadb root password"
      ignore_errors: yes
      mysql_user:
        login_user: root
        login_password: ''
        user: root
        password: "{{ mysql_root }}"
        host_all: yes
      tags:
        - lamp
        - lamp-wordpress

    - name: "Lampstack - removing anonymous users"
      mysql_user:
        login_user: root
        login_password: "{{ mysql_root }}"
        user: ''
        host_all: yes
        state: absent
      tags:
        - lamp
        - lamp-wordpress

    - name: "Lampstack - creating database"
      mysql_db:
        login_user: root
        login_password: "{{ mysql_root }}"
        name: "{{ wordpress_database }}"
        state: present
      tags:
        - lamp
        - lamp-wordpress

    - name: "Lampstack - creating database user"
      mysql_user:
        login_user: root
        login_password: "{{ mysql_root }}"
        name: "{{ wordpress_user }}"
        password: "{{ wordpress_password }}"
        host: localhost
        priv: "{{ wordpress_database }}.*:ALL"
      tags:
        - lamp
        - lamp-wordpress

    - name: "Wordpress - Download tar"
      get_url:
        url: https://wordpress.org/wordpress-4.8.9.tar.gz
        dest: /tmp/wordpress.tar.gz
      tags:
        - lamp-wordpress

    - name: "Wordpress - Extracting file in tmp"
      unarchive:
        src: /tmp/wordpress.tar.gz
        dest: /tmp/
        remote_src: yes
      tags:
        - lamp-wordpress

    - name: "Wordpress - copying files to document root"
      shell: "cp -r /tmp/wordpress/* /var/www/html/{{ domain }}/"
      tags:
        - lamp-wordpress

    - name: "Wordpress - uploading wp-config.php"
      template:
        src: wp-config.php.j2
        dest: "/var/www/html/{{ domain }}/"
      tags:
        - lamp-wordpress

    - name: "Wordpress - changing ownership"
      shell: 'chown -R {{ domain_owner }}:{{ domain_group }} /var/www/html/{{domain}}/'
      tags:
        - lamp-wordpress

    - name: "Wordpress - changing permission of files to 644"
      shell: 'find /var/www/html/{{domain}}/ -type f -exec chmod 644 {} \;'
      tags:
        - lamp-wordpress

    - name: "Wordpress - Chaning Directory Permission to 755"
      shell: 'find /var/www/html/{{domain}} -type d -exec chmod 755 {} \;'
      tags:
        - lamp-wordpress

    - name: "Wordpress - removing /tmp/wordpress.tar.gz /tmp/wordpress"
      file:
        path: "{{item}}"
        state: absent
      with_items:
        - /tmp/wordpress.tar.gz
        - /tmp/wordpress
      tags:
        - lamp-wordpress
 ```
        
 Template files
 --------------
 
 #vim virtualhost.j2
 ```
<virtualhost *:80>
  servername {{domain}}
  documentroot /var/www/html/{{domain}}
  directoryindex index.php index.html info.html info.php
</virtualhost>
```


#vim wp-config.php.j2
```

<?php

define( 'DB_NAME', '{{wordpress_database}}' );
define( 'DB_USER', '{{wordpress_user}}' );
define( 'DB_PASSWORD', '{{wordpress_password}}' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8' );
define( 'DB_COLLATE', '' );

define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );

$table_prefix = 'wp_';

define( 'WP_DEBUG', false );
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', dirname( __FILE__ ) . '/' );
}

require_once( ABSPATH . 'wp-settings.php' );
```

       
