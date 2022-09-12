# lemp-wordpress-ansible

# package.vars
```
---
packages: 
  - nginx
  - php7.4
  - php7.4-mysql
  - php7.4-fpm
```

# nginx.vars
```
---
lport: 80
proot: "/var/www/html"
domain_name: "example.timetrav.online"
nuser: "www-data"
ngroup: "www-data"
```

# nvhost.conf.j2
```
server {

listen {{ lport }};
listen [::]:{{ lport }};
root {{ proot }}/{{ domain_name }};

index index.php index.html index.htm index.nginx-debian.html;

server_name {{ domain_name }};

location / {
  try_files $uri $uri/ =404;
}

location ~ \.php$ {
  include snippets/fastcgi-php.conf;

  # Nginx php-fpm sock config:
  fastcgi_pass unix:/run/php/php7.4-fpm.sock;
}
} 
```

# mysqle.vars
```
---
mysqlrpe: "mysqlroot123pass"
mysql_add_db: "wordpressdb"
mysql_add_user: "wordpress"
mysql_add_pass: "wordpress@123"

```

# vim wordpress.vars
```
---
wp_url: "https://wordpress.org/wordpress-5.8.4.tar.gz"

```






# my.cnf.tmpl

```
[client]
user=root
password={{ mysqlrpe }}

```






# wp-config.php.tmpl

```
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the web site, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * Database settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://wordpress.org/support/article/editing-wp-config-php/
 *
 * @package WordPress
 */

// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', '{{ mysql_add_db }}' );

/** Database username */
define( 'DB_USER', '{{ mysql_add_user }}' );

/** Database password */
define( 'DB_PASSWORD', '{{ mysql_add_pass }}' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication unique keys and salts.
 *
 * Change these to different unique phrases! You can generate these using
 * the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}.
 *
 * You can change these at any point in time to invalidate all existing cookies.
 * This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );

/**#@-*/

/**
 * WordPress database table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://wordpress.org/support/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', false );

/* Add any custom values between this line and the "stop editing" line. */



/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';

```


```

---
- name: "Installing and Configuring Nginx webServer on Ubuntu"
  hosts: ubuntu
  become: true
  vars_files:
    - package.vars
    - nginx.vars
    - mysqle.vars
    - wordpress.vars

  tasks:
    
    - name: "Installing wanted pacakages"
      apt:
        update_cache: yes
        name: "{{ packages }}"
        state: present

    - name: check if domain configuration {{ domain_name }}.conf  already exist on server"
      stat:
        path: /etc/nginx/sites-available/{{ domain_name }}.conf
      register: check_ex_status

    - name: check if documentroot {{ proot }}/{{ domain_name }} already exist on server"
      stat:
        path: /etc/nginx/sites-available/{{ domain_name }}.conf
      register: check_ex2_status

    - name: "removing {{ domain_name }}.conf , {{ proot }}/{{ domain_name }} from server. files already exist!"
      when: check_ex_status.stat.exists == true or check_ex2_status.stat.exists == true
      file:
        state: absent
        path: "{{ item }}"
      with_items:
          - "{{ proot }}/{{ domain_name }}"
          - "/etc/nginx/sites-available/{{ domain_name }}.conf"


    - name: "copying nginx vhosts for domain {{ domain_name }}"
      template:
        src: nvhost.conf.j2
        dest: "/etc/nginx/sites-available/{{ domain_name }}.conf"
      register: vcopy_status

    - name: "creating symlink to sites-enabled for vhost {{ domain_name }}.conf"
      file:
        state: link
        src: "/etc/nginx/sites-available/{{ domain_name }}.conf"
        dest: "/etc/nginx/sites-enabled/{{ domain_name }}.conf"

    - name: "creating documentroot for domain {{ domain_name }}"
      file:
        state: directory
        path: "{{ proot }}/{{ domain_name }}"
        owner: "{{ nuser }}"
        group: "{{ ngroup }}"


    - name: "Restarting nginx,php service with conditions"
      when: vcopy_status.changed == true
      service:
        name: nginx
        state: restarted
        enabled: yes

    - name: "Installing maraidb-server packages"
      apt:
        name: 
          - mariadb-server
          - python3-mysqldb
        state: present

    - name: "Restarting and enabling maraiadb services"
      service: 
        name: mariadb
        state: restarted
        enabled: yes


    - name: check if mysqlpassword file exist
      stat:
        path: /root/.my.cnf
      register: check_status1

    - debug:
        var: check_status1   
    
    - name: "Setting root password for mariadb if empty"
      when: check_status1.stat.exists == false
      mysql_user:
        login_user: "root"
        login_password: ""
        name: "root"
        password: "{{ mysqlrpe }}"
        host_all: yes
      register: check_status2
      
    - name: "Copying template if root password exist"
      when: check_status2.changed == true
      template:
        src: my.cnf.tmpl
        dest: /root/.my.cnf
        mode: 0644
      register: check_status3

    - name: "Post restart mariadb services after copying template file"
      when: check_status3.changed == true
      service: 
        name: mariadb
        state: restarted


    - name: "Deleting anonymous mysql users"
      mysql_user:
        login_user: "root"
        login_password: "{{ mysqlrpe }}"
        user: ""
        state: absent

    - name: "Deleting database {{ mysql_add_db }} if exist"
      mysql_db:
        login_user: "root" 
        login_password: "{{ mysqlrpe }}"
        state: absent
        name: "{{ mysql_add_db }}"

    - name: "Deleting database user: {{ mysql_add_user }} if exist"
      mysql_user:
        login_user: "root" 
        login_password: "{{ mysqlrpe }}"
        state: absent
        name: "{{ mysql_add_user }}"
        host: "%"
        

    - name: "Creating additional database {{ mysql_add_db }} for wordpress site {{ domain_name }}"
      mysql_db:
        login_user: "root" 
        login_password: "{{ mysqlrpe }}"
        state: present
        name: "{{ mysql_add_db }}"
        
    - name: "Creating additional database user: {{ mysql_add_user }} for wordpress site {{ domain_name }}"
      mysql_user:
        login_user: "root" 
        login_password: "{{ mysqlrpe }}"
        state: present
        name: "{{ mysql_add_user }}"
        host: "%"
        password: "{{ mysql_add_pass }}"
        priv: "{{ mysql_add_db }}.*:ALL"

    - name: "Downloading wordpress tar file: /tmp/wordpress.tar.gz"
      get_url: 
        url: "{{ wp_url }}"
        dest: /tmp/wordpress.tar.gz

    - name: "Extracting conetnts on /tmp/wordpress.tar.gz"
      unarchive:
        src: /tmp/wordpress.tar.gz
        dest: /tmp
        remote_src: true

    - name: "Copying file contents on /tmp/wordpress to {{ proot }}/{{ domain_name }}"
      copy:
        remote_src: true
        src: /tmp/wordpress/
        dest: "{{ proot }}/{{ domain_name }}"
        owner: "{{ nuser }}"
        group: "{{ ngroup }}"

    - name: "Creating a wp-config.php from template wp-config.php.tmpl"
      template:
        src: wp-config.php.tmpl
        dest: "{{ proot }}/{{ domain_name }}/wp-config.php"
        owner: "{{ nuser }}"
        group: "{{ ngroup }}"

    - name: "Post Installation cleanup"
      file:
        state: absent
        path: "{{ item }}"
      with_items:
          - /tmp/wordpress
          - /tmp/wordpress.tar.gz

    - name: "Post Installation Restart"
      service: 
        state: restarted
        name: "{{ item }}"
      with_items:
          - nginx
          - php7.4-fpm
          - mariadb

    - debug:
        msg: "Your Nginx+wordpress Installation is complete, you can continue wordpress set up by calling {{ domain_name }} on the browser."
        
```
