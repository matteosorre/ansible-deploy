# Redis Role

Ansible role to install and configure Redis server for Django-Q task queue.

## Description

This role installs Redis server and the Python redis client library, configured specifically for use with Django-Q, a multiprocessing task queue for Django.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
# Redis server configuration
needs_redis_server: true              # Install Redis server
redis_bind: 127.0.0.1                 # Bind address
redis_port: 6379                      # Redis port
redis_maxmemory: 256mb                # Maximum memory allocation
redis_maxmemory_policy: allkeys-lru   # Eviction policy

# Django-Q supervisord configuration (optional)
configure_django_q: false             # Set to true to auto-configure supervisord for Django-Q
django_q_workers: 4                   # Number of Django-Q workers
django_project_path: ""               # Absolute path to Django project (e.g., /var/www/myproject)
django_virtualenv_path: ""            # Absolute path to virtualenv (e.g., /var/www/myproject/venv)
django_q_user: www-data               # User to run Django-Q as
django_q_log_file: /var/log/django_q.log  # Log file path
```

## What This Role Does

1. Installs Redis server (if `needs_redis_server` is true)
2. Installs Python redis client library
3. Configures Redis with specified settings
4. Ensures Redis service is started and enabled on boot
5. **Optionally** installs and configures supervisord to manage Django-Q workers (if `configure_django_q` is true)

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  roles:
    - redis
```

With custom variables:
```yaml
- hosts: all
  roles:
    - role: redis
      vars:
        redis_bind: 0.0.0.0  # Allow external connections
        redis_maxmemory: 512mb
```

**With Django-Q supervisord auto-configuration:**
```yaml
- hosts: all
  roles:
    - role: redis
      vars:
        configure_django_q: true
        django_project_path: /var/www/myproject
        django_virtualenv_path: /var/www/myproject/venv
        django_q_user: www-data
```

This will automatically:
- Install and configure Redis
- Install supervisor
- Create supervisord configuration for Django-Q
- Start Django-Q workers automatically
- Ensure workers restart on failure or reboot

## Django-Q Configuration

After running this role, you need to configure Django-Q in your Django project:

### 1. Install Django packages
```bash
pip install django-q redis
```

### 2. Add to Django settings.py
See `templates/DJANGO_Q_SETUP.md` for complete configuration examples.

Quick example:
```python
INSTALLED_APPS = [
    # ...
    'django_q',
]

Q_CLUSTER = {
    'name': 'DjangORM',
    'workers': 4,
    'redis': {
        'host': '127.0.0.1',
        'port': 6379,
        'db': 0,
    }
}
```

### 3. Run migrations and start cluster
```bash
python manage.py migrate
python manage.py qcluster
```

## Templates

- `django_q_settings.py.j2`: Django-Q configuration template
- `DJANGO_Q_SETUP.md`: Complete setup guide for Django-Q with Redis

## License

MIT

## Author Information

This role was created for deploying Django applications with Django-Q task queue.
