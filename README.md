Ansible Django deployment role
==============================

This role installs the skeleton of a Django web application infrastructure that can easily be upgraded.

The Django application is running behind a web server (Apache or Nginx), and with uWSGI.

In more details, the role:

* it configures the web server (currently NGinx and Apache2 supported), with all the options needed to communicate with
  uWSGI and with the SSL certificates,
* it configures uWUSGI for serving the Django application
* it sets up the directories skeleton and appropriate ACLs for the application to run,
* it creates an **upgrade version** script that ease the deployment of new versions of the application. This script can be called with a few parameters
  and takes care of snapshots, backups, etc (suitable for eg. an Atlassian Bamboo deployment target)
* it takes a good care of the file containing the **secrets** password/variables needed for running the Django application
  (salt, access to the authentication server, etc) that should not go into the repository of the application.

In more details, the deployment scripts takes care of the following:

* creates a new folder following the version name given
* installs a new ``virtualenv`` in that folder and installs the dependencies of the project. It can optionally source a `requirements.txt` file (see ``django_pip_install_requirements``)
* binds uWSGI to the ``virtualenv``
* deflate the sources in that folder
* creates a snapshot of the database
* performs the migration of the database
* collects the static files
* modifies the ACLs such that the ``uwsgi`` user can consume the files
* modifies the ACLs of the secret file
* modifies the `current` top-level link to the newly created folder
* adds optional commands to the uWSGI startup to start eg. Celery at the same time as the server
* and finally restarts the uWSGI daemon to take into account the changes

### Limitations
* Currently only sqlite is supported in the deployment procedure, but I accept any PR that overcome this limitation (the state of the DB should be
  saved during a new deployment)
* the deployment script does not support downgrade

Requirements
------------

This works only on Ubuntu but can easily be adapted for other platforms. Prior to running this role,
prepare the packages that needs to be installed on the target machine with the role `ansible-django-webapp-host`.

Role Variables
--------------

|variable|default|meaning|
|----------|---------|---------|
|websiteproject | **required** | the name of this project. This will define the root location of the application.|
|url| **required** | the url of the application|
|django_source_folder |  | the location of the app in the source folder (to be compared to the location of the `wsgi.py` file). This is mainly used in case we need to deflate static files|
|django_archives_to_deflate | | additional list of files to deflate after deflating the source archive. This can be used to deflate for instance bootstrap from within the archive|
|django_dependencies| | additional django dependencies for the project. This will install the newest (or specified) versions of the dependencies into a dedicated virtualenv|
|django_deployment_root_folder| **required** | root folder for the django applications|
|django_settings_folder|**required**| location of the `wsgi` wrapper, needed for `uWSGI` configuration|
|django_settings_file| **required** | the name of the settings file for this deployment, relative to the django_settings_folder.|
|django_webapplication_additional_environment_settings| | a dictionary containing additional environment variables to be set for running the application|
|django_revision_file_pattern | **required** | pattern of the deployment tarballs. This pattern may contain additional environment variables such as `${bamboo_planRepository_revision}` that indicates the revision generating this tarball. This is used in order to identify the file to extract during the
deployment script.|
|django_pip_install_requirements | `False` | If `True` installs the requirements given by a `requirement.txt` file inside the deployed archive within the virtual environment. |
|application_additional_packages| `[]` | additional packages to install that are specific to the application (if your web application needs eg. `rabbitmq`)|

* the `django_dependencies` should contain a stable set of dependencies. `Django` itself is not specified in case a specific version should be installed.
* the `django_settings_file` contains the settings that will be executed on the system. It better be configured to source an external configuration/secret file
  (see section below)


### Web server and uWSGI configurations

**Global**

|variable|default|meaning|
|----------|---------|---------|
|webserver| **required** | the webserver in front of uWSGI|
|site_certificate| **required** | the certificate of the website|
|site_key|**required**| the key of the certificate|

**uWSGI**

|variable|default|meaning|
|----------|---------|---------|
|uwsgi_number_workers| 2 | number of workers of the uWSGI|
|uwsgi_user|`www-data` | the user running uWSGI|
|uwsgi_group| `www-data` | the group of the user running uWSGI|
|uwsgi_additional_options| `[]` | additional options passed to the uWSGI configuration file|

**NGinx**

|variable|default|meaning|
|----------|---------|---------|
|nginx_binary_name| `/usr/sbin/nginx` | name of the NGinx binary: used for testing the configuration when NGinx is used|
|nginx_user  | `www-data` | user of the Nginx instance: this is used for the ACLs on the certificate file|
|nginx_group | `www-data` | group of the NGinx daemon: this is used for the ACLs on the certificate file|

**Apache**

|variable|default|meaning|
|----------|---------|---------|
|apache2_check_binary_name | `a2ensite` | same as `nginx_binary_name` but Apache specific|
|apache2_user| `www-data`| user running the Apache2 daemon|
|apache2_group| `www-data`| group of the user running the Apache2 daemon|

**Additional variables**

|variable|default|meaning|
|----------|---------|---------|
|site_proxy|| proxy to use/expose when the firewall is limiting the connexion to the outside (during deployment, eg. on a DMZ for running PIP like commands)|


**SSL Certificate**

Depending on the `webserver`, the certificate will be stored in:
* `/etc/nginx/ssl/` for NGinx
* `/etc/apache2/ssl/` for Apache

and named after the name of the project `websiteproject`. The access to those files will be limited to only the user running Apapche or Nginx.

### Extras

The `webserver` should be the same as for the `ansible-django-webapp-prepare-host` role.
Most of the uWSGI configuration inherit from the default one provided by Ubuntu

The `uwsgi_additional_options` can be used to set up for instance other daemons that should start at the same time
as the web application. In case of `Celery` for instance, a possible option would be

```
uwsgi_additional_options:
  - "smart-attach-daemon2 = %(pidfile)_celery %(virtualenv)/bin/celery --app={{django_settings_folder}} worker -l info --pidfile=%(pidfile)_celery"
```

where `django_settings_folder` is set to the application server settings folder, and the `%(virtualenv)` is already set by the configuration
scripts.


How to deploy a new version of your application
-----------------------------------------------

The deployment part just needs to perform the following for upgrading to a new version:

* scp the new archive to the `$app_root/versions`
* call the `$app_root/deployment.sh` script with the following parameters (in order):
  * the environment (not used)
  * the current release name
  * the previous release version
  * the revision of the repository that generated the source tarball: this goes to the environment variable `${bamboo_planRepository_revision}` inside the
    script and may be used in conjunction with the `django_revision_file_pattern` variable (see below)

Recipe for creating a deployable django-app
-------------------------------------------

The scope of this module is to have a deployment environment for Django applications as well as a script that allows the deployment. The design is the following:

* the host machine decides where to store the application. This is given by the variable `django_deployment_root_folder` and all applications go in that folder.

* several settings should be supported by the application, and the settings file used for the deployment (`django_settings_file`)
  should be able to source from an external file. The following example
  * expects the `DJANGO_SETTINGS_SECRET_FILE` environment variable to be set: this is required by our uWSGI script
  * sources the content of the file pointed by the environment variable and extracts the secrets

        secret_file = os.environ['DJANGO_SETTINGS_SECRET_FILE']
        secret_dict = {}
        with open(secret_file) as secret:
            exec(secret.read(), secret_dict)

  * the subsequent reference to secrets can be done the following way, instead of hardcoding it into the source files

        # SECURITY WARNING: keep the secret key used in production secret!
        SECRET_KEY = secret_dict['DJANGO_ENV_SECRET_KEY']

  * in the playbook/inventory for this application, those secrets can be stored like this:

        django_environment_variables:
            DJANGO_ENV_SECRET_KEY: "{{ vault_coffeine_django_secret_key }}"
            DJANGO_CROWD_USER: "{{ vault_coffeine_crowd_user }}"
            DJANGO_CROWD_PASS: "{{ vault_coffeine_crowd_passwd }}"

    and can then be read from the Ansible vault.

Dependencies
------------

This role is highly correlated with `ansible-django-webapp-prepare-host` that is used to setup the machine (installation of webserver, services, python etc).
The call of this role is not mandatory though.

Example Playbook
----------------

* Create an inventory entry, for instance for the `code_doc` project (located here: https://bitbucket.org/renficiaud/code_doc), like this:

      [code_doc-production]
      prodserver.my.localnet # indicates where the production of code_doc will go

      [code_doc-staging]
      vm-staging.my.localnet # indicates what server will host the staging environment of this project

      [code_doc:children]
      code_doc-production
      code_doc-staging

* Some specific configuration for `prodserver` and `vm-staging` (like passwords in ansible-vaults, the `webserver`, `url`, `django_deployment_root_folder` ...) can then be stored in their respective inventory file `host_vars/prodserver.my.localnet` or `host_vars/vm-staging.my.localnet`

* The playbook for preparing the target machine for this application (`code_doc`) can for instance be:

      # code_doc django application

      - hosts: code_doc
        vars:

          # general name of the project
          websiteproject: code_doc

          # url of the project
          url: code_doc.my.localnet

          # certificate and private key:
          # the certificate can be in clear
          site_certificate: "{{ vault_code_doc_ssl_certificate }}"
          # the key should be in the vault
          site_key: "{{ vault_code_doc_ssl_certificate_key }}"

          # Dango project specific variables, this is because we need to deflate
          # Boostrap
          django_source_folder: code_doc

          # deflate archives
          django_archives_to_deflate:
            - "{{ django_source_folder }}/static/bootstrap-3.2.0-dist.zip"

          # folder containing the wsgi.py file
          django_settings_folder: www

          # django settings for the deployed application
          # this one is for the production environment, but this variable can go to the inventory
          # file as well
          django_settings_file: "{{ django_settings_folder }}.settings.code_doc_production"

          # this is the file that are copied from bamboo: this can contain the environment variables
          # that are inherited when the deployment script is called
          django_revision_file_pattern: code_doc-r${bamboo_planRepository_revision}.tar.bz2

          # django dependencies of the project, installed in a separate virtual environment
          # for each new deployment
          django_dependencies:
            - django
            - django-markdown
            - pillow

          # Reading values from the secret. Those variables are populated by uWSGI upon
          # startup of the web application, and are hence available in the settings file
          # of Django.
          django_environment_variables:
            DJANGO_ENV_SECRET_KEY: "{{ vault_code_doc_django_secret_key }}"
            DJANGO_CROWD_USER: "{{ vault_code_doc_crowd_user }}"
            DJANGO_CROWD_PASS: "{{ vault_code_doc_crowd_passwd }}"

        # those are the secrets of the application
        vars_files:
          - vault_vars/code_doc
        roles:
        - ansible-django-webapp

* Finally, running ansible like this:

      ansible-playbook djangoapp_code_doc.yml -i my-inventory  -l code_doc-production -vvvv -k

License
-------

BSD

Author Information
------------------

Any comments on the Ansible, PR or bug reports are welcome from the corresponding Github project.
