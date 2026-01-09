# Post Installation

Run the following command to create a superuser:

```bash
kubectl -n <namespace> exec -it deployment/taiga-back -- python manage.py createsuperuser
```