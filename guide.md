# Laravel Nginx Setup Guide

This guide shows how to deploy a Laravel application behind Nginx using the configuration in `default.conf`.

## Prerequisites

- Ubuntu or another Linux server
- Nginx installed
- PHP-FPM installed and running on `php8.3-fpm`
- Composer installed
- A Laravel project ready to deploy

## 1. Upload the project

Move the Laravel project to the server location used by the web server.

```bash
sudo mv /home/ubuntu/site /var/www/subdomain.domain.com
```

If you use a different domain, replace `subdomain.domain.com` with your own folder name.

## 2. Set ownership and permissions

Give Nginx access to the project files and make Laravel writable directories usable.

```bash
sudo chown -R www-data:www-data /var/www/subdomain.domain.com
sudo chmod -R 775 /var/www/subdomain.domain.com/storage
sudo chmod -R 775 /var/www/subdomain.domain.com/bootstrap/cache
```

## 3. Create the Nginx site config

Create a new Nginx site file in `/etc/nginx/sites-available/` and paste the contents of `default.conf` into it.

```bash
sudo nano /etc/nginx/sites-available/laravel-app
```

The provided config does the following:

- listens on port 80
- serves the Laravel `public` directory
- redirects extra domains to the main domain
- passes PHP requests to `/var/run/php/php8.3-fpm.sock`
- blocks direct access to hidden files
- increases upload size to `20M`

## 4. Enable the site

Link the config into `sites-enabled` so Nginx loads it.

```bash
sudo ln -s /etc/nginx/sites-available/laravel-app /etc/nginx/sites-enabled/
```

If a previous config exists with the same name, remove it first before creating the symlink.

## 5. Test and reload Nginx

Always test the configuration before restarting the service.

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 6. Install Laravel dependencies

Go into the project directory and install production dependencies.

```bash
cd /var/www/subdomain.domain.com
composer install --optimize-autoloader --no-dev
```

## 7. Configure Laravel environment

If the project does not already have a `.env` file, copy the example file and generate the application key.

```bash
cp .env.example .env
php artisan key:generate
```

Update the `.env` file with your database, cache, queue, and app URL settings.

## 8. Finish Laravel setup

Run any remaining Laravel setup commands your project needs.

```bash
php artisan migrate --force
php artisan storage:link
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

## Example Nginx config

Use the following config as a reference. It is the same structure as `default.conf`.

```nginx
server {
	listen 80;
	listen [::]:80;
	server_name subdomain.domain.com;
	root /var/www/subdomain.domain.com/public;

	add_header X-Frame-Options "SAMEORIGIN";
	add_header X-Content-Type-Options "nosniff";
	client_max_body_size 20M;

	index index.php;
	charset utf-8;

	location / {
		try_files $uri $uri/ /index.php?$query_string;
	}

	location = /favicon.ico { access_log off; log_not_found off; }
	location = /robots.txt  { access_log off; log_not_found off; }

	error_page 404 /index.php;

	location ~ \.php$ {
		fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
		fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
		include fastcgi_params;
	}

	location ~ /\.(?!well-known).* {
		deny all;
	}
}
```

## Common issues

- If Nginx returns `502 Bad Gateway`, check that PHP-FPM is running and the socket path is correct.
- If Laravel shows permission errors, re-check ownership of `storage` and `bootstrap/cache`.
- If the site does not load, verify that the `root` points to the Laravel `public` directory.
- If config changes do not apply, re-run `sudo nginx -t` and reload Nginx.
