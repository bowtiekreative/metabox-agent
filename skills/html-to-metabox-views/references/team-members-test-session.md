# Session Reference: Team Members CPT — HTML → MetaBox Pipeline Test

Created Jun 16, 2026. This can be used as a template for future CPTs.

## Workflow Executed

1. Analyzed team-card.html → identified team_member CPT, fields (role, bio, photo, social_links group), skill taxonomy
2. Created plugin at `wp-content/plugins/team-members/team-members.php`
3. Created Local JSON at `theme/twentytwentyfive/mb-json/team-fields.json` (4 fields + cloneable social_links group)
4. Created Twig template at `theme/twentytwentyfive/mb-views/team-card.twig`
5. Created test script at `test-team-view.php` — simulates Twig rendering and outputs structured verification

## Commands Used

```bash
# Create plugin
mkdir -p /var/www/html/wp-content/plugins/team-members/
write_file path=/var/www/html/wp-content/plugins/team-members/team-members.php ...

# Create Local JSON
write_file path=/var/www/html/wp-content/themes/twentytwentyfive/mb-json/team-fields.json ...

# Activate
wp --allow-root --path=/var/www/html plugin activate team-members
wp --allow-root --path=/var/www/html rewrite flush --hard

# Create terms
wp --allow-root --path=/var/www/html term create skill "PHP" --porcelain
wp --allow-root --path=/var/www/html post term set <ID> skill PHP WordPress React

# Create post (publish explicitly!)
wp --allow-root --path=/var/www/html post create \
  --post_type=team_member \
  --post_title="Jane Doe" \
  --post_status=publish \
  --meta_input='{"role": "Lead Developer", "bio": "..."}'

# Set group field data via PHP (NOT wp-cli)
php -r '
require_once "wp-load.php";
update_post_meta(9, "social_links", [
    ["platform" => "twitter", "url" => "https://twitter.com/janedoe"],
]);
'

# Fix plugin file permissions
chmod -R 755 /var/www/html/wp-content/plugins/team-members/

# Test
cd /var/www/html && php test-team-view.php
```

## Verification

After setup, verify with:
```bash
# REST API check
curl -s "https://<domain>/wp-json/wp/v2/types" | grep -o '"slug":"[^"]*"'

# wp-cli post count
wp --allow-root --path=/var/www/html post list --post_type=team_member --posts_per_page=5

# MetaBox field values
wp --allow-root --path=/var/www/html eval '
echo rwmb_meta("role", [], 9);
'
```