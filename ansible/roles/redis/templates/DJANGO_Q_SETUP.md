# Django-Q with Redis - Configuration Guide

## Required Django Packages

Add to your Django project's requirements.txt:
```
django-q
redis
```

## Django Settings Configuration

Add the following to your Django `settings.py`:

```python
# Add 'django_q' to INSTALLED_APPS
INSTALLED_APPS = [
    # ... your other apps
    'django_q',
]

# Django-Q Configuration
Q_CLUSTER = {
    'name': 'DjangORM',
    'workers': 4,                    # Number of worker processes
    'recycle': 500,                  # Recycle workers after this many tasks
    'timeout': 60,                   # Task timeout in seconds
    'compress': True,                # Compress task packages
    'save_limit': 250,               # Number of successful tasks to keep in database
    'queue_limit': 500,              # Maximum number of tasks in queue
    'cpu_affinity': 1,               # CPU affinity for workers
    'label': 'Django Q',
    'redis': {
        'host': '127.0.0.1',         # Redis host (use {{ redis_bind }} in template)
        'port': 6379,                # Redis port (use {{ redis_port }} in template)
        'db': 0,                     # Redis database number
        'password': None,            # Redis password if required
        'socket_timeout': None,
        'charset': 'utf-8',
        'errors': 'strict',
        'unix_socket_path': None
    }
}
```

## Running Django-Q

After configuration, run migrations:
```bash
python manage.py migrate
```

Start the Django-Q cluster:
```bash
python manage.py qcluster
```

## Usage Examples

### Schedule a task
```python
from django_q.tasks import async_task, schedule
from django_q.models import Schedule

# Run a task asynchronously
async_task('myapp.tasks.my_function', arg1, arg2, kwarg1='value')

# Schedule a recurring task
schedule('myapp.tasks.daily_report',
         schedule_type=Schedule.DAILY,
         repeats=-1)  # -1 means repeat forever
```

### Create a task function
```python
# myapp/tasks.py
def my_function(arg1, arg2, kwarg1=None):
    # Your task logic here
    print(f"Processing: {arg1}, {arg2}, {kwarg1}")
    return "Task completed"
```

## Monitoring

Access Django admin to monitor tasks:
- Successful tasks: `/admin/django_q/success/`
- Failed tasks: `/admin/django_q/failure/`
- Scheduled tasks: `/admin/django_q/schedule/`

## Production Deployment

For production, use supervisord to manage the qcluster process. The `supervisord` role in this project can be configured to manage Django-Q workers.

Example supervisord configuration:
```ini
[program:django_q]
command=/path/to/venv/bin/python /path/to/manage.py qcluster
directory=/path/to/project
user=www-data
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/django_q.log
```
