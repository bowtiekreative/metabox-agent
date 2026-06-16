---
name: wordpress-local
description: "Set up a local WordPress instance with Docker for testing MetaBox fields, Views, shortcodes, and CPTs"
version: 1.0.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [wordpress, docker, local, testing, meta-box, wp-cli, metabox]
    related_skills: [metabox-custom-fields, metabox-views, metabox-cpt, wordpress-code]
---

# Local WordPress Setup for Testing — Skill Guide

Set up a disposable WordPress + MySQL environment with Docker for testing MetaBox code before deploying to production.

## Prerequisites

- Docker and Docker Compose installed
- Internet connection for pulling images
- Ports 8080 (WordPress) and 3306 (MySQL) available

## Docker Compose Setup

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - ./wp-content/plugins:/var/www/html/wp-content/plugins
      - ./wp-content/themes:/var/www/html/wp-content/themes
      - ./wp-content/uploads:/var/www/html/wp-content/uploads

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

### Start WordPress

```bash
docker compose up -d
```

Wait ~30 seconds for WordPress to initialize, then visit `http://localhost:8080`.

## Quick Setup Script

```bash
#!/bin/bash
# setup-wp.sh — Quick local WordPress for MetaBox testing

WP_DIR="${1:-./wordpress-test}"
mkdir -p "$WP_DIR/wp-content/plugins" "$WP_DIR/wp-content/themes" "$WP_DIR/wp-content/uploads"

cat > "$WP_DIR/docker-compose.yml" << 'EOF'
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    ports: ["8080:80"]
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - ./wp-content/plugins:/var/www/html/wp-content/plugins
      - ./wp-content/themes:/var/www/html/wp-content/themes
      - ./wp-content/uploads:/var/www/html/wp-content/uploads
  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - db_data:/var/lib/mysql
volumes:
  db_data:
EOF

cd "$WP_DIR"
docker compose up -d
echo "Waiting 30s for WordPress..."
sleep 30
echo "WordPress running at http://localhost:8080"
```

## Installing wp-cli

```bash
# Download wp-cli
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp

# Set up completion
wp --info
```

## Install MetaBox Plugin

```bash
# Inside the WordPress container
docker compose exec wordpress bash -c "
  # Install MetaBox from WordPress.org (free version)
  wp plugin install meta-box --activate

  # Or copy a premium version from host
  # (place zip in ./wp-content/plugins/ and activate)
"
```

Or from host via wp-cli:

```bash
# Install MetaBox Lite
wp --path=./wp-content plugin install meta-box --activate

# Install MB Custom Post Type
wp plugin install mb-custom-post-type --activate
```

## Testing Your Code

### 1. Mount your PHP code

Place your `functions.php` additions or plugin code:

```bash
# As a plugin (recommended for testing)
cat > wp-content/plugins/metabox-test/metabox-test.php << 'PLUGIN'
<?php
/**
 * Plugin Name: MetaBox Test
 * Description: Test MetaBox custom fields
 */
add_filter('rwmb_meta_boxes', function ($meta_boxes) {
    $meta_boxes[] = [
        'title'      => 'Test Fields',
        'post_types' => 'post',
        'fields'     => [
            ['name' => 'Test Field', 'id' => 'test_field', 'type' => 'text'],
        ],
    ];
    return $meta_boxes;
});
PLUGIN

wp plugin activate metabox-test
```

### 2. Create test content

```bash
# Create a post with meta
wp post create \
  --post_type=post \
  --post_title="Test Post" \
  --meta_input='{"test_field": "Hello MetaBox!"}'

# Verify
wp post meta get <post_id> test_field
```

### 3. Test your View/Shortcode/Template

```bash
# Create a page with the shortcode
wp post create \
  --post_type=page \
  --post_title="Test Shortcode Page" \
  --post_content='[rwmb_meta id="test_field"]'

# Verify rendering
wp post view <page_id>
```

## Cleanup

```bash
# Stop and remove everything
docker compose down -v
rm -rf ./wp-content ./db_data docker-compose.yml
```

## Quick Reference

| Action | Command |
|--------|---------|
| Start WP | `docker compose up -d` |
| Stop | `docker compose down` |
| Full reset | `docker compose down -v && docker compose up -d` |
| WP CLI | `docker compose exec wordpress wp <command>` |
| Shell in WP | `docker compose exec wordpress bash` |
| View logs | `docker compose logs -f wordpress` |
| Copy plugin | `cp plugin.zip wp-content/plugins/ && docker compose exec wordpress wp plugin activate <name>` |

## Troubleshooting

### 3306 port conflict
Change MySQL port mapping:
```yaml
db:
  ports: ["3307:3306"]
```

### Connection refused on start
Wait 30-60s for MySQL to initialize before installing plugins.

### Memory issues
Limit resources in docker-compose:
```yaml
wordpress:
  deploy:
    resources:
      limits:
        memory: 512M
```