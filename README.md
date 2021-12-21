# Ansible Role Apache 2.4

An Ansible Role that installs Apache 2.4 on RHEL/CentOS/OL 8.

 * 1.0.0 initial release
 * 1.0.1 install php_mod by default, so php files are actually processed
 * 1.1.0 set php_mod as option. Include more default modules. Update readme
 * 1.1.1 set servername to supress warning during configtest
 * 1.1.2 set alias also for ssl vhost
 * 1.1.3 set httpd.conf as a template
 * 1.1.4 set comment for Listen <port> when there is no vhost defined
 * 1.1.5 improved logrotate script

## Good 2 know

 * Ensure the httpd packages are available for the host
 * Ensure the docroot path exists! Otherwise, httpd will fail. Do this with a `pre-task`, see the example playbook below
 * Ensure the ssl files are placed if required, see example playbook
 * A configcheck is always executed before reloading Apache. If the configcheck fails, a reload is not executed
 * The welcome file is replaced with an empty file, due to it's restoration when apache is updated

## Role Variables

Installing a specific version is possible. This role will check if the package is available before installing.
Setting a specific version is not recommended. When apache is updated to a higher version, and this role is run again, it will downgrade apache. So either set version leave version as 2.4. Or make sure you update the version tag when apache is updated as well.

```yaml
apache_version: 2.4.6-89
```

The IP address and ports on which apache should be listening. Useful if you have another service (like a reverse proxy) listening on port 80 or 443 and need to change the defaults.

```yaml
apache_listen_ip: "*"
apache_listen_port: 80
apache_listen_port_ssl: 443
```

You can add or override global Apache configuration settings in the role-provided vhosts file (assuming `apache_create_vhosts` is true) using this variable. By default it only sets the DirectoryIndex configuration.

```yaml
apache_global_vhost_settings: |
  DirectoryIndex index.php index.html
  # another global vhost line here
```

Add a set of properties per virtualhost, including `servername` (required), `documentroot` (required), `allow_override` (optional: defaults to the value of `apache_allow_override`), `options` (optional: defaults to the value of `apache_options`), `serveradmin` (optional), `serveralias` (optional) and `extra_parameters` (optional: you can add whatever additional configuration lines you'd like in here).

```yaml
apache_vhosts:
  # Additional optional properties: 'serveradmin, serveralias, extra_parameters'.
  - servername: local.dev
    documentroot: /var/www/html
```

Here's an example using `extra_parameters` to add a RewriteRule to redirect all requests to the `www.` site:
```yaml
- servername: www.local.dev
  serveralias: local.dev
  documentroot: /var/www/html
  extra_parameters: |
    RewriteCond %{HTTP_HOST} !^www\. [NC]
    RewriteRule ^(.*)$ http://www.%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/var/run/php-fpm/www-data.sock|fcgi://localhost/"
    </FilesMatch>
```

For vhosts with SSL support:

```yaml
apache_vhosts_ssl:
  - servername: solvinity.com
    documentroot: /var/www/secure/html
    certificate_file: /path/of/the.crt
    certificate_key_file: /path/of/the.key
    certificate_chain_file: /path/to/crt_chain.crt
    extra_parameters: |
      RewriteCond %{HTTP_HOST} !^www\. [NC]
      RewriteRule ^(.*)$ http://www.%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
      <FilesMatch \.php$>
          SetHandler "proxy:unix:/var/run/php-fpm/www-data.sock|fcgi://localhost/"
      </FilesMatch>
```

Other SSL directives can be managed with other SSL-related role variables.
```yaml
apache_ssl_protocol: "All -SSLv2 -SSLv3"
apache_ssl_cipher_suite: "AES256+EECDH:AES256+EDH"
```

The SSL protocols and cipher suites that are used/allowed when clients make secure connections to your server. These are secure/sane defaults, but for maximum security, performand, and/or compatibility, you may need to adjust these settings.

```yaml
apache_allow_override: "All"
apache_options: "-Indexes +FollowSymLinks"
```

The default values for the `AllowOverride` and `Options` directives for the `documentroot` directory of each vhost. A vhost can overwrite these values by specifying `allow_override` or `options`.

See the `defaults/main.yml` for a complete list of default mods which are enabled.

Enable or disable mods:

```yaml
apache_additional_modules:
  - rewrite.load
  - ssl.load
apache_mods_disabled: []
```

If you would like to only create SSL vhosts when the vhost certificate is present (e.g. when using Let’s Encrypt), set `apache_ignore_missing_ssl_certificate` to `false`.

```yaml
apache_ignore_missing_ssl_certificate: false
```

## .htaccess-based Basic Authorization

If you require Basic Auth support, you can add it either through a custom template, or by adding `extra_parameters` to a VirtualHost configuration, like so:

```yaml
extra_parameters: |
  <Directory "/var/www/password-protected-directory">
    Require valid-user
    AuthType Basic
    AuthName "Please authenticate"
    AuthUserFile /var/www/password-protected-directory/.htpasswd
  </Directory>
```

To password protect everything within a VirtualHost directive, use the `Location` block instead of `Directory`:

```yaml
<Location "/">
  Require valid-user
  ....
</Location>
```

You would need to generate/upload your own `.htpasswd` file in your own playbook.

## Logrotate

By default, a daily logrotate is done, and the logs are kept on disk for 14 days.
The file is placed in `/etc/logrotate.d/httpd`.
Set this variable with:

```yaml
httpd_logrotate: |
  my settings
  other settings
  rotate 1
```

## Example playbook

```yaml
---
- hosts: webserver
  become: true
  roles:
    - apache
  pre_tasks:
    - name: ensure apache vhosts documentroot directories exists
      file:
        state: directory
        path: "{{ item.documentroot }}"
        mode: '0775'
        owner: someone # or when using php, "{{ php_fpm_owner }}"
        group: somegroup # or when using php "{{ php_fpm_group }}"
      loop: "{{ apache_vhosts }}"

    - name: ensure apache ssl vhosts documentroot directories exists
      file:
        state: directory
        path: "{{ item.documentroot }}"
        mode: '0775'
        owner: someone # or when using php, "{{ php_fpm_owner }}"
        group: somegroup # or when using php"{{ php_fpm_group }}"
      loop: "{{ apache_vhosts_ssl }}"

    - name: ensure key and certs are placed
      copy:
        src: "ssl/{{ item.src }}"
        dest: "/etc/pki/tls/{{ item.dest }}/"
        mode: "{{ item.mode }}"
        owner: root
        group: root
      loop:
        - { src: my.crt, dest: 'certs', mode: '0644' }
        - { src: my-chain.crt, dest: 'certs', mode: '0644' }
        - { src: my.key, dest: 'private', mode: '0600' }
```
