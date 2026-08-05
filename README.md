This is a collection of Ansible roles that automate the deployment of a Django project on a Linux server.

# How to use the roles with your project

* Create a directory `ansible/` in the root of your repository containing an Ansible `hosts` inventory. Here you could have just one line with the SSH public address of your target server.
* Create a `ansible/templates/` subdirectory containing a `env.j2` file that will be copied to the server as a Django .env file.
* Create a `ansible/vars/` subdirectory containing a `master.yaml` file that will be used to setup the Ansible variables used by roles.
  You could have more than one file, one for each GIT branch, if you need to support more than one environment (production, staging, etc.).

An example `env.j2` file is this:

```
SERVER_NAME={{ server_name }}
DATABASE_URL={{ database_url }}
STATIC_ROOT={{ static_dir }}
MEDIA_ROOT={{ media_dir }}
```

An example `master.yaml` file is this:

```
---
ansible_user: root
project_name: yourproject
server_name: yourproject.example.com
home_dir: /home/ubuntu
virtualenv_path: "{{ home_dir }}/.virtualenvs/{{ project_name }}"
root_dir: "{{ home_dir }}/{{ project_name }}"
static_dir: "{{ root_dir }}/static"
media_dir: "{{ root_dir }}/media"
commands_log_path: "{{ root_dir }}/commands_log"
requirements_dir: "{{ root_dir }}/requirements"
django_dir: "{{ root_dir }}/django"
django_project: project
needs_db_server: true
dbhost: localhost
dbtype: postgres
dbname: django_yourproject
dbuser: django_yourproject
dbpassword: "{{ lookup('env', 'DJANGO_DB_PASSWD') }}"
database_url: "{{ dbtype }}://{{ dbuser }}:{{ dbpassword }}@{{ dbhost }}/{{ dbname }}"
project_repo_path: "../../{{ project_name }}"
python_requirements_path: "{{ project_repo_path }}/requirements/"
django_src_path: "{{ project_repo_path}}/django/"
django_env_template_path: "{{ project_repo_path }}/ansible/templates/env.j2"
deploy_environment: production
uwsgi_workers: 1
git_rev: "{{ lookup('env', 'CIRCLE_SHA1') }}"
```

# How to deploy automatically using CircleCI

Create a `.circleci/config.yml` file in the root of your repository:

```
# See: https://circleci.com/docs/2.0/configuration-reference
version: 2.1
jobs:
  deploy:
    executor: python/default
    steps:
      - checkout
      - run: |
          git clone https://github.com/baxeico/ansible-deploy.git
      - python/install-packages:
          pip-dependency-file: ansible-deploy/requirements/circleci.txt
          pkg-manager: pip
      - run: |
          cd ansible-deploy/ansible/
          ansible-playbook -i ../../yourproject/ansible/hosts -e "@../../yourproject/ansible/vars/<< pipeline.git.branch >>.yaml" deploy_django.yaml
orbs:
  python: circleci/python@1.3.2
workflows:
  main:
    jobs:
      - deploy:
          filters:
            branches:
              only:
                master
```

Push it on your repository and then setup the project on CircleCI.

CircleCI will start building right away, but the first run will fail because we didn't yet setup the authentication part.

Let's setup that:

* Generate a private/public SSH with `ssh-keygen -m PEM -t rsa -C "deploy@circleci.com"`. The email address is arbitrary;
* In CircleCI project settings go in "SSH Keys" > "Additional SSH Keys" and click "Add Key". Copy the private key generated at step before;
* Now login to your target server and add the public key in the `.ssh/authorized_keys` file for the user that will be used by Ansible for deploy.
* Add the DJANGO_DB_PASSWD environment variable to your CircleCi project configuration. That will be used to setup the database used by Django.

# Mailpit role

The `mailpit` role installs and configures [Mailpit](https://mailpit.axe.email/), a lightweight email testing tool with an SMTP server and a web UI for inspecting captured emails. This is useful for development and staging environments where you don't want to send real emails.

## Usage

Use the `deploy_mailpit.yaml` playbook:

```
ansible-playbook -i ../../yourproject/ansible/hosts deploy_mailpit.yaml
```

Or add the `mailpit` role to your existing playbook:

```yaml
roles:
  - postgresqlclient
  - django-nginx-uwsgi
  - mailpit
```

Then configure your Django application to use Mailpit as the email backend:

```
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=localhost
EMAIL_PORT=1025
```

The Mailpit web interface will be available at `http://your-server:8025`.

## Configuration variables

| Variable | Default | Description |
|---|---|---|
| `mailpit_version` | `v1.21.8` | Mailpit release version |
| `mailpit_listen_smtp` | `0.0.0.0:1025` | SMTP listen address |
| `mailpit_listen_http` | `127.0.0.1:8025` | Web UI listen address (localhost by default) |
| `mailpit_max_messages` | `500` | Maximum number of messages to store |
| `mailpit_user` | `mailpit` | System user for the service |
| `mailpit_group` | `mailpit` | System group for the service |
| `mailpit_nginx` | `false` | Enable nginx reverse proxy for the web UI |
| `mailpit_nginx_server_name` | `""` | Server name for nginx (e.g. `mail.example.com`) |
| `mailpit_nginx_ssl` | `false` | Enable HTTPS with Let's Encrypt certificates |

## Nginx reverse proxy

To expose the Mailpit web UI via nginx with HTTPS (e.g. `https://mail.spoc.online`), set these variables:

```yaml
mailpit_nginx: true
mailpit_nginx_server_name: mail.spoc.online
mailpit_nginx_ssl: true
```

This will configure nginx as a reverse proxy with SSL using Let's Encrypt certificates.
Make sure the certificates are already available at `/etc/letsencrypt/live/{{ mailpit_nginx_server_name }}/` (e.g. via `certbot`).
