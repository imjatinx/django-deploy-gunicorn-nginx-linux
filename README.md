# django-deploy-gunicorn-nginx-linux
This is a complete quick guide to deploy a django application with apache2 web server using gunicorn interface.

To deploy a **Django** application on your VPS, you typically use **Gunicorn** as the application server and **Nginx** (or Apache) as the reverse proxy. Here's a step-by-step guide on deploying Django on your VPS, assuming you're using **Apache2** (since you used Apache earlier) and want to set it up with a domain and SSL.

### Prerequisites:
- A VPS with root access (DigitalOcean in your case).
- Apache2 installed.
- A domain name pointing to your VPS.
- SSL with Certbot (Let's Encrypt) installed.

### 1. **Install Python and Virtual Environment**
Make sure you have Python installed, along with `pip` and `virtualenv` to isolate your project dependencies.

```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv
```

### 2. **Clone Your Django App to the VPS**
SSH into your VPS and navigate to the directory where you want to deploy your application, such as `/var/www/your-django-app`:

```bash
cd /var/www
git clone https://github.com/your-username/your-django-app.git
cd your-django-app
```

### 3. **Set Up a Virtual Environment**
Create and activate a virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate
```

Install the necessary dependencies (including Django, Gunicorn, and any other requirements for your app):

```bash
pip install -r requirements.txt
```

### 4. **Configure Django Settings for Production**
- **Update `settings.py`**: Modify your `ALLOWED_HOSTS` to include your domain name and server IP:

```python
ALLOWED_HOSTS = ['your-domain.com', 'www.your-domain.com', 'your-server-ip']
```

- **Set up Static and Media files**:
  
In `settings.py`, configure the paths for serving static files:
```python
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

Collect the static files:
```bash
python manage.py collectstatic
```

### 5. **Set Up Gunicorn (WSGI Server)**
Gunicorn is a WSGI server that will serve your Django application.

1. Install Gunicorn:

```bash
pip install gunicorn
```

2. Test Gunicorn with your Django app:

```bash
gunicorn --workers 3 your_django_project.wsgi:application
```

If the app runs successfully, you can proceed to create a systemd service for Gunicorn.

### 6. **Set Up Gunicorn as a Systemd Service**
To keep your Django app running, create a systemd service file for Gunicorn:

1. **Create a Gunicorn service file**:

```bash
sudo nano /etc/systemd/system/gunicorn.service
```

2. **Add the following configuration**:

```ini
[Unit]
Description=gunicorn daemon for Django application
After=network.target

[Service]
User=your-username
Group=www-data
WorkingDirectory=/var/www/your-django-app
ExecStart=/var/www/your-django-app/venv/bin/gunicorn --workers 3 --bind unix:/var/www/your-django-app/gunicorn.sock your_django_project.wsgi:application

[Install]
WantedBy=multi-user.target
```

3. **Start and enable Gunicorn**:

```bash
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```

Check the status of Gunicorn to ensure it's running:

```bash
sudo systemctl status gunicorn
```

### 7. **Configure Apache2 as a Reverse Proxy**
You’ll now configure Apache2 to act as a reverse proxy to Gunicorn. We will use `mod_proxy` to forward requests from Apache to Gunicorn.

1. **Enable necessary Apache modules**:

```bash
sudo a2enmod proxy proxy_http proxy_uwsgi proxy_wstunnel
sudo systemctl restart apache2
```

2. **Create a Virtual Host Configuration for your domain**:

```bash
sudo nano /etc/apache2/sites-available/your-domain.conf
```

Add the following configuration:

```apache
<VirtualHost *:80>
    ServerName your-domain.com
    ServerAlias www.your-domain.com

    ProxyPass /static/ !
    Alias /static/ /var/www/your-django-app/static/

    <Directory /var/www/your-django-app/static>
        Require all granted
    </Directory>

    ProxyPass / unix:/var/www/your-django-app/gunicorn.sock|http://localhost/
    ProxyPassReverse / unix:/var/www/your-django-app/gunicorn.sock|http://localhost/

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

This configuration tells Apache to serve the static files directly and pass all other requests to Gunicorn via the socket.

3. **Enable your site and restart Apache**:

```bash
sudo a2ensite your-domain.conf
sudo systemctl restart apache2
```

### 8. **Set Up SSL with Certbot (Optional)**
If you want to enable HTTPS using Let's Encrypt, use Certbot to automatically configure SSL for your domain.

1. Install Certbot (if not already installed):

```bash
sudo apt install certbot python3-certbot-apache
```

2. Obtain an SSL certificate and configure Apache:

```bash
sudo certbot --apache -d your-domain.com -d www.your-domain.com
```

Certbot will automatically update your Apache configuration to serve your Django app securely over HTTPS.

### 9. **Verify Deployment**
Visit your domain (`http://your-domain.com` or `https://your-domain.com` if using SSL) and verify that your Django app is running correctly.

### Summary:
- **Gunicorn**: Use it as the WSGI server for your Django app.
- **Apache**: Configure it as a reverse proxy to serve your app.
- **Certbot**: Use it to set up SSL for secure HTTPS access.
- **Static Files**: Serve them using Apache while routing dynamic requests to Gunicorn.

Let me know if you need further clarification or help with any specific step!
