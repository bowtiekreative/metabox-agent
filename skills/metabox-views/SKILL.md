---
name: metabox-views
description: "MB Views — Twig-based frontend templates with MetaBox fields, location rules, REST API limitations, and PHP integration"
version: 1.2.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [meta-box, views, twig, templates, wordpress, metabox, rest-api]
    related_skills: [metabox-custom-fields, metabox-shortcodes, wordpress-code]
---

# MetaBox Views — Skill Guide

MB Views enables **Twig-based front-end templates** using Meta Box fields, post fields, site settings, user fields, and query fields.

## REST API & XML-RPC — Creating Views Programmatically

### `mbv` namespace has READ-ONLY internal routes only
- The `/mbv` namespace registers in the REST API index, but only has hidden routes (`show_in_index=false`): `/mbv/views`, `/mbv/meta-boxes`, `/mbv/relationships` — all GET-only, permission-checked via `manage_options`
- You **cannot** create, read, update, or delete Views via the REST API

### `mb-views` post type is not REST-exposed
- Views are stored as custom post type `mb-views` in the database
- This CPT has `show_in_rest` disabled
- `wp/v2/mb-views` returns `rest_no_route` (404)
- Supports: `title`, `revisions` only (no `editor`, so `post_content` is NOT managed via the WordPress admin editor)

### ✅ Creating MB Views via XML-RPC

Even though REST is blocked, **XML-RPC works** because `mb-views` uses standard `capability_type => 'post'` with `map_meta_cap => true`:

**Step 1: Create the View**
```bash
curl -X POST -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?>
<methodCall>
  <methodName>wp.newPost</methodName>
  <params>
    <param><value><int>1</int></value></param>
    <param><value><string>admin</string></value></param>
    <param><value><string>APP_PASSWORD</string></value></param>
    <param><value><struct>
      <member>
        <name>post_type</name>
        <value><string>mb-views</string></value>
      </member>
      <member>
        <name>post_title</name>
        <value><string>Team Members Grid</string></value>
      </member>
      <member>
        <name>post_content</name>
        <value><string>TWIG_TEMPLATE_CODE</string></value>
      </member>
      <member>
        <name>post_status</name>
        <value><string>publish</string></value>
      </member>
    </struct></value></param>
  </params>
</methodCall>' \
  "https://example.com/xmlrpc.php"
# Returns post ID (string)
```

**Step 2: Set View meta settings**
The Twig template lives in `post_content`. Additional settings are stored as post meta:
- `mode` — set to `"shortcode"` to enable `[mbv id="slug"]` usage
- `type` — set to `"shortcode"` for shortcode mode, `"singular"` for single-post display
- `code` — alternative storage (NOT used for rendering — actual rendering reads from `post_content`)
- `singular_locations` — location rules for singular/archive type views

Set these via `wp.editPost` with `custom_fields`:
```bash
curl -X POST -H "Content-Type: text/xml" \
  -d '... <member><name>custom_fields</name>...' \
  "https://example.com/xmlrpc.php"
```

**Step 3: Use the shortcode**
Once `mode=shortcode` and the View is published, use `[mbv id="<post_name>"]` anywhere (page content, widget, etc.). The `post_name` is auto-generated from the title.

### MB Views Template Storage

The TwigLoader reads template source from **`$post->post_content`** (line 29 of `TwigLoader.php`):
```php
$source = $post->post_content;
```
Additional parts:
- `$post->post_excerpt` — wrapped in `<style>` tags and appended
- `$post->post_content_filtered` — wrapped in `<script>` tags and appended

So for a View with embedded CSS: put Twig in `post_content`, CSS in `post_excerpt`.

### MB Views Twig Syntax

The `mb` object available in Twig templates is an instance of `MBViews\TwigProxy`. It provides:

**Available methods on `mb`:**
| Method | Purpose | Example |
|--------|---------|---------|
| `mb.get_posts(args)` | WP_Query wrapper | `mb.get_posts({post_type: "team_member", posts_per_page: -1})` |
| `mb.get_post(id)` | Get single post | `mb.get_post(9)` |
| `mb.get_term(id)` | Get term object | `mb.get_term(3)` |
| `mb.get_user(id)` | Get user object | `mb.get_user(1)` |
| `mb.get_terms(args)` | get_terms wrapper | `mb.get_terms({taxonomy: "skill"})` |
| `mb.get_users(args)` | get_users wrapper | `mb.get_users({role: "author"})` |
| `mb.map(field_id)` | Render map field | `mb.map("location")` |
| `mb.checkbox(value)` | Yes/no display | `mb.checkbox(post.featured, "Yes", "No")` |
| `mb.post_comments()` | Comments template | `{{ mb.post_comments() }}` |
| **Any PHP function** | Proxied via `__call` | `mb.get_the_date()`, `mb.the_content()` |

**Accessing custom field data on post objects:**
The `Post` renderer merges all MetaBox field values into accessible properties via `__get`:
```twig
{{ member.role }}          {# reads the "role" custom field #}
{{ member.bio }}           {# reads the "bio" custom field #}
{{ member.post_title }}    {# standard WordPress field #}
{{ member.social_links }}  {# group/cloneable field — serialized array #}
```

**Loop pattern with custom fields:**
```twig
{% for member in mb.get_posts({post_type: "team_member", posts_per_page: -1}) %}
  <h3>{{ member.post_title }}</h3>
  <p class="role">{{ member.role }}</p>
  <div class="bio">{{ member.bio }}</div>
{% else %}
  <p>No team members found.</p>
{% endfor %}
```

### What IS available via REST
- Meta Box Builder (`mbb`) namespace has full CRUD: `/mbb/save`, `/mbb/json-data`, `/mbb/fields-ids`, `/mbb/field-types`, `/mbb/field-html`
- `mbb/set-json-data` — syncs JSON field files to/from DB (see `metabox-custom-fields` skill for details)
- Meta Box core (`meta-box/v1`) has no custom routes
- MB Frontend Submission (`mbfs`) has no custom routes
- MB Blocks (`mb-blocks/v1`) has minimal routes
- MB Relationships (`mb-relationships/v1`) has `generate` + `save` endpoints

## When MB Views is NOT available

MetaBox AIO (free) does **not** include MB Views — it's a premium extension. Check availability by:
1. `curl /wp-json/wp/v2/plugins` — look for "Meta Box AIO"
2. Check post type registration: `curl /wp-json/wp/v2/types` — look for `mb-views`
3. Check DB: `SELECT * FROM wp_postmeta WHERE meta_key LIKE '%mb_views%'`
4. Admin menu: look for "Meta Box → Views" in sidebar

## Alternative Approaches When Views Unavailable

### 1. JavaScript Client-Side Rendering (used in this session)
- Create a static page with embedded JS that fetches CPT data from REST API
- Works with any theme (especially block themes like Twenty Twenty-Five)
- Skills taxonomy needs mapping: fetch `/wp/v2/skill?per_page=100` and map IDs→names
- Example pattern:
  ```js
  const [membersRes, skillsRes] = await Promise.all([
    fetch('/wp-json/wp/v2/team_member?_embed'),
    fetch('/wp-json/wp/v2/skill?per_page=100')
  ]);
  ```
- Set the page as `show_on_front: "page"` via `/wp/v2/settings`

### 2. PHP Shortcode / Template Approach
- Create a small plugin with `add_shortcode()` that queries CPTs via `WP_Query`
- Use `rwmb_get_value()` to retrieve MetaBox field data
- Add shortcode to page content or override theme template

### 3. Block Theme Front Page
- Twenty Twenty-Five has a `home.html` template with Blog heading + posts query loop
- Set static front page via `/wp/v2/settings`: `show_on_front: "page"`, `page_on_front: <id>`
- Block themes respect these settings but `home.html` template takes over for blog index
- Verify with `curl ?nocache=$(date +%s)` or browser — `web_extract` may return stale cached content

## MB Views Template Structure
```
wp-content/themes/your-theme/
├── mb-json/          # Local JSON field definitions (auto-loaded)
│   ├── team-fields.json
│   └── directory-fields.json
└── mb-views/         # Twig templates (require MB Views premium)
    └── team-card.twig
```

## Reference Files
- `references/front-page-js-rendering.md` — Full working example of client-side JS rendering of CPT data on a static front page, with skills taxonomy mapping

## Pitfalls
- **Cache confusion**: After setting static front page, `web_extract` may still show blog index. Use `?nocache=` param or browser/vision to verify
- **No REST CRUD for Views**: You must use admin UI or direct DB/WP-CLI to create Views; no programmatic API via REST
- **Block theme templates**: `home.html` can override front page settings if not properly configured
- **JS rendering and SEO**: Client-side rendering means search engines may not index team member content server-side; use PHP approach for SEO-critical sites