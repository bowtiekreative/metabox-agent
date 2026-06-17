---
name: html-to-metabox-views
description: "Convert HTML to MetaBox Views (Twig) — create CPT, Local JSON fields, Twig template, and test on local WordPress in one pipeline"
version: 1.1.0
metadata:
  hermes:
    tags: [html, metabox, views, twig, wordpress, cpt, local-json]
    related_skills: [metabox-custom-fields, metabox-views, metabox-cpt, wordpress-local]
---

# HTML → MetaBox Views Pipeline

Convert any HTML into a working MetaBox View with CPT, JSON fields, and Twig template, tested on local WordPress.

## Workflow

### 1. Analyze the HTML
Identify:
- **CPT**: What content type? (team_member, portfolio, directory)
- **Fields**: What data does it display? (text, image, select, group, etc.)
- **Taxonomies**: What categories/tags? (skill, category)
- **Structure**: Cards? Lists? Single items?

### 2. Create the CPT Plugin
Write a plugin at `/var/www/html/wp-content/plugins/<name>/<name>.php`:
```php
<?php
add_action('init', function () {
    register_post_type('<cpt_slug>', [
        'labels'  => [...],
        'public'  => true,
        'has_archive' => true,
        'menu_icon'   => 'dashicons-...',
        'supports'    => ['title', 'editor', 'thumbnail'],
        'rewrite'     => ['slug' => '<slug>'],
        'show_in_rest' => true,
    ]);

    // Optional: register taxonomy
    register_taxonomy('<tax_slug>', '<cpt_slug>', [
        'hierarchical' => false,
        'public'       => true,
        'rewrite'      => ['slug' => '<slug>'],
        'show_in_rest' => true,
    ]);
});
```

### 3. Create Local JSON Fields
Write to `/var/www/html/wp-content/themes/twentytwentyfive/mb-json/<name>.json`:

**IMPORTANT:** The meta box group MUST include an `id` field. Without it, MetaBox Builder throws `Undefined array key "id"` and the admin UI won't recognize the group.

```json
{
  "$schema": "https://schemas.metabox.io/field-group.json",
  "id": "my_group_id",             ← REQUIRED for Builder compatibility
  "title": "...",
  "post_types": "<cpt_slug>",
  "context": "normal",
  "style": "default",
  "fields": [
    { "name": "Field", "id": "field_id", "type": "text" }
  ],
  "modified": <timestamp>
}
```

### 4. Create Twig Template — Two Approaches

#### Option A: Local JSON file-based (for static theme template)
Write to `/var/www/html/wp-content/themes/twentytwentyfive/mb-views/<name>.twig`:
```twig
{% if post.photo %}
  <img src="{{ post.photo.full_url }}" alt="{{ post.post_title }}">
{% endif %}`
<h3>{{ post.post_title }}</h3>
<p>{{ post.role }}</p>
<p>{{ post.bio }}</p>
{% for link in post.social_links %}
  <a href="{{ link.url }}">{{ link.platform }}</a>
{% endfor %}
{% for skill in post.terms('skill') %}
  <span>{{ skill.name }}</span>
{% endfor %}
```

#### Option B: XML-RPC View creation (Dashboard-managed View)
For Views that should appear in the **Meta Box → Views** admin UI, create them as `mb-views` CPT posts via XML-RPC instead of static `.twig` files:

```bash
# 1. Create the View post
VIEW_ID=$(curl -s -X POST -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?>
<methodCall>
  <methodName>wp.newPost</methodName>
  <params>
    <param><value><int>1</int></value></param>
    <param><value><string>admin</string></value></param>
    <param><value><string>APP_PASSWORD</string></value></param>
    <param><value><struct>
      <member><name>post_type</name><value><string>mb-views</string></value></member>
      <member><name>post_title</name><value><string>Team Grid</string></value></member>
      <member><name>post_content</name><value><string>TWIG_TEMPLATE</string></value></member>
      <member><name>post_status</name><value><string>publish</string></value></member>
    </struct></value></param>
  </params>
</methodCall>' \
  "https://example.com/xmlrpc.php" | grep -oP '<string>\K[^<]+')

# 2. Set mode=shortcode so [mbv id="slug"] works
# Use wp.editPost with custom_fields
```

**Key differences from file-based approach:**
- Twig template goes in `post_content` (NOT in a `code` meta field)
- CSS goes in `post_excerpt` (auto-wrapped in `<style>` by Renderer)
- `post_name` (slug) is auto-generated from the title
- Set `mode` and `type` meta to `"shortcode"` for shortcode usage
- Use `[mbv id="<post_name>"]` anywhere to render
- Access custom fields as properties: `{{ member.role }}`, `{{ member.bio }}`

**MB Views Twig syntax reference:**
```twig
{% for member in mb.get_posts({post_type: "team_member", posts_per_page: -1}) %}
  <h3>{{ member.post_title }}</h3>
  <p>{{ member.role }}</p>           {# custom field via __get #}
  <div>{{ member.bio }}</div>         {# custom field via __get #}
{% else %}
  <p>No members found.</p>
{% endfor %}
```

### 5. (Optional) Import to MetaBox Builder Database

To make field groups visible and editable in the **Meta Box → Custom Fields** admin UI:

```bash
# Ensure mb-json/ is writable by www-data
chown -R www-data:www-data wp-content/themes/twentytwentyfive/mb-json/
chmod -R 755 wp-content/themes/twentytwentyfive/mb-json/

# Import each field group
curl -X POST -u "admin:pass" \
  -H "Content-Type: application/json" \
  -d '{"id": "my_group_id", "use": "json"}' \
  "https://example.com/wp-json/mbb/set-json-data"
# → {"success": true}

# Verify
curl -s -u "admin:pass" \
  "https://example.com/wp-json/mbb/json-data" | jq '.[].id'
```

See `metabox-custom-fields` skill for details on the `mbb/set-json-data` endpoint.

### 6. Test
Run test script: `cd /var/www/html && php test-<name>-view.php`

The test script should:
- Fetch all posts of the CPT
- Output structured verification (emojis for humans)
- Output the rendered HTML for inspection
- Verify all fields, groups, relationships, and taxonomies

## Pitfalls

- **Group field data**: Use `update_post_meta(ID, 'field', $array)` directly in PHP, NOT via wp-cli `wp post meta update` or `--meta_input` which double-serializes the array into a string that `rwmb_meta()` cannot parse.
- **wp-cli post status**: `wp post create` defaults to `draft`, not `publish`. Always pass `--post_status=publish` explicitly, otherwise the post won't appear on the front-end or in REST API queries.
- **Plugin file permissions in Docker**: Files created as root get `-rw-------` (600) permissions. The web server runs as `www-data` and cannot read them, causing silent plugin activation failure. Always run `chmod -R 755` on plugin files after creation.
- **`get_posts` for new CPTs**: Must include `'post_status' => 'any', 'suppress_filters' => true` to find recently created posts. Without these, freshly published CPT posts may return 0 results.
- **MB Views activation**: MB Views is bundled inside `meta-box-aio/vendor/meta-box/mb-views/` but requires the `meta_box_aio` option's `extensions` array to include `'mb-views'`. If it's missing, enable it via `update_option('meta_box_aio', ['extensions' => [...full list...]])` in PHP.
- **MetaBox AIO module discovery**: To list all available AIO modules, glob `dirname(WP_PLUGIN_DIR . '/meta-box-aio/vendor/meta-box') . '/*'` and filter to directories only. The AIO option stores them by slug.
- **Taxonomy slugs**: After creating terms via wp-cli, flush rewrite rules with `wp rewrite flush --hard`.
- **JSON sync**: Increase `modified` timestamp in the JSON file to trigger re-sync detection in MetaBox admin.
- **Cache verification**: After setting a static front page, `web_extract` may return stale cached content showing the blog index instead of the page. Use `curl ?nocache=$(date +%s)` or `browser_vision` (browser screenshot + analysis) to confirm the live rendering instead of relying on `web_extract` text output.**: Always verify with `get_post_meta(ID, 'key', true)` + `var_export()` to catch double-serialization issues.

## Verification

- [ ] CPT registered (`wp post-type list`)
- [ ] JSON fields visible (`rwmb_meta('field', [], ID)`)
- [ ] Taxonomy terms assigned (`wp_get_post_terms(ID, 'tax')`)
- [ ] Group/social links render correctly
- [ ] HTML output matches source HTML structure
- [ ] No PHP warnings in test output
- [ ] Front page renders correctly (verify with `?nocache=` or browser, not just `web_extract`)