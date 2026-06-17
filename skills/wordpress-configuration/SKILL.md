---
name: wordpress-configuration
description: "Configure WordPress production sites — PHP upload limits, memory, OPcache tuning, Elementor/WooCommerce pre-reqs, module verification, web server reload."
version: 1.0.0
tags:
  - wordpress
  - php
  - elementor
  - opcache
  - uploads
  - performance
  - apache
platforms: [linux]
metadata:
  hermes:
    triggers:
      - "increase upload size"
      - "wordpress settings"
      - "elementor requirements"
      - "php limits"
      - "wordpress memory limit"
      - "opcache"
      - "worpdress setup"
      - "wp config"
    related_skills: [wordpress-code, wordpress-local]
---

# WordPress Production Configuration

Configure a WordPress site for production — upload limits, PHP memory, OPcache, and plugin-specific requirements (Elementor, WooCommerce, MetaBox).

## Step 1: Check Current PHP Settings

```bash
php -r "
echo 'upload_max_filesize: ' . ini_get('upload_max_filesize') . PHP_EOL;
echo 'post_max_size: ' . ini_get('post_max_size') . PHP_EOL;
echo 'memory_limit: ' . ini_get('memory_limit') . PHP_EOL;
echo 'max_execution_time: ' . ini_get('max_execution_time') . PHP_EOL;
echo 'max_input_time: ' . ini_get('max_input_time') . PHP_EOL;
"
```

## Step 2: Create PHP Override

In Docker WordPress containers, add a `.ini` file to the PHP conf.d directory:

```bash
cat > /usr/local/etc/php/conf.d/uploads.ini << 'EOF'
upload_max_filesize = 256M
post_max_size = 300M
memory_limit = 512M
max_execution_time = 300
max_input_time = 300
EOF
```

**Recommended values by use case:**

| Use Case | upload_max | post_max | memory_limit | max_exec_time |
|---|---|---|---|---|
| Default WordPress | 64M | 64M | 256M | 120 |
| **Elementor ready** | **256M** | **300M** | **512M** | **300** |
| WooCommerce heavy | 128M | 128M | 256M | 300 |
| Large media site | 512M | 512M | 768M | 300 |

## Step 3: Verify PHP Modules Required by Plugins

**Elementor requires:**

```bash
php -m | grep -iE 'gd|imagick|zip|intl|curl|mbstring|dom|xml|json'
```

Expected: all should show output. Common missing modules can be installed via:

```bash
apt-get install -y php-gd php-imagick php-zip php-intl php-curl php-mbstring php-dom php-xml
```

## Step 4: WordPress Constants (wp-config.php)

Add these **after** the `WP_DEBUG` line, before "stop editing":

```php
// Increase memory limits
define( 'WP_MEMORY_LIMIT', '512M' );
define( 'WP_MAX_MEMORY_LIMIT', '512M' );

// Safety: limit revisions
define( 'WP_POST_REVISIONS', 10 );
```

## Step 5: OPcache Tuning

Create or update OPcache config (in `/usr/local/etc/php/conf.d/opcache-recommended.ini` or similar):

```ini
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.revalidate_freq=2
opcache.max_wasted_percentage=15
opcache.validate_timestamps=1
```

**For Elementor sites:** Higher OPcache values significantly improve admin panel responsiveness since Elementor compiles CSS/JS on the fly.

## Step 6: Restart Web Server

```bash
# Apache
apache2ctl graceful

# Nginx + PHP-FPM
nginx -s reload && systemctl reload php*-fpm
```

Verify settings took effect:

```bash
php -r "
echo 'upload: ' . ini_get('upload_max_filesize') . PHP_EOL;
echo 'post: ' . ini_get('post_max_size') . PHP_EOL;
echo 'memory: ' . ini_get('memory_limit') . PHP_EOL;
"
```

## Step 7: Verify WordPress Sees the Changes

Go to **WordPress Admin → Media → Add New** — the upload limit should now show the new value. Or install a "Server Info" style plugin to confirm.

For Elementor specifically, check **Elementor → System Info** — it will report PHP memory limit and upload size constraints if they're too low.

## Pitfalls

- **In Docker WordPress containers**, finding the PHP config location: check `php --ini` — it often has no loaded `php.ini` and only uses `conf.d/` drop-in files. Creating a new `.ini` in `conf.d/` is the right approach.
- **upload_max_filesize vs post_max_size**: `post_max_size` must be >= `upload_max_filesize` since file uploads are part of the POST body. Rule of thumb: post_max = upload_max + ~20%.
- **WP_MEMORY_LIMIT vs WP_MAX_MEMORY_LIMIT**: `WP_MEMORY_LIMIT` is the frontend limit; `WP_MAX_MEMORY_LIMIT` is for the admin dashboard. Both should be set.
- **Elementor on low memory**: If Elementor editor shows "Something went wrong" or white screens on save, insufficient memory is the #1 cause — increase to 512M.
- **OPcache validation**: After changing OPcache settings, restart the web server — OPcache settings are read at startup, not per-request.
- **Hostinger shared hosting**: Some hosts cap upload limits in their own nginx config regardless of PHP settings. If changes don't stick, check with support or look for a custom `user.ini` in the web root.

## Verification Checklist

- [ ] PHP config values match intended targets
- [ ] WordPress admin shows the new upload limit
- [ ] All required PHP modules present
- [ ] Elementor System Info shows no red warnings
- [ ] OPcache enabled and configured
- [ ] Apache/Nginx reloaded