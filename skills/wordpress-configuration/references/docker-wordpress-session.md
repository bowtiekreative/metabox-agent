# Session Reference — Docker WordPress Elementor Config

Applied Jun 16, 2026 on a Docker WordPress container (Apache, PHP 8.x).

## PHP Config Applied

Created `/usr/local/etc/php/conf.d/uploads.ini`:

```ini
upload_max_filesize = 256M
post_max_size = 300M
memory_limit = 512M
max_execution_time = 300
max_input_time = 300
```

## OPcache Applied

Updated `/usr/local/etc/php/conf.d/opcache-recommended.ini`:

```ini
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.revalidate_freq=2
opcache.max_wasted_percentage=15
opcache.validate_timestamps=1
opcache.file_cache=/tmp/opcache-file
opcache.file_cache_only=0
```

## WordPress Constants Added

Inserted in `/var/www/html/wp-config.php` after `WP_DEBUG`:

```php
define( 'WP_MEMORY_LIMIT', '512M' );
define( 'WP_MAX_MEMORY_LIMIT', '512M' );
define( 'WP_POST_REVISIONS', 10 );
```

## Verified PHP Modules

All Elementor-required modules present: gd, imagick, zip, intl, curl, mbstring, dom, SimpleXML, xml, xmlreader, xmlwriter

## Container Details

- **Base:** WordPress Docker image (Apache + PHP)
- **PHP config path:** `/usr/local/etc/php/conf.d/` (no base `php.ini` loaded)
- **Web server:** Apache 2 (PID 1 as `apache2 -DFOREGROUND`)
- **WordPress path:** `/var/www/html/`
- **DB:** MySQL 8.4 on container host `db` (172.16.1.2)