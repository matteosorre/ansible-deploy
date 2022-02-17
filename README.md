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
env_template_path: "{{ project_repo_path }}/ansible/templates/env.j2"
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
      - run: |
          mkdir ~/.ssh && chmod 700 ~/.ssh
          echo -e "Host github.com\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config
          git clone --branch << pipeline.git.branch >> git@github.com:baxeico/ansible-deploy.git
          git clone --branch << pipeline.git.branch >> git@github.com:youruser/yourproject.git
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
