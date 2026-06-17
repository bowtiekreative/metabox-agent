# MB Views XML-RPC & Twig Reference

From session that built team member Views on `wordpress-plac.srv620544.hstgr.cloud`.

## XML-RPC View Creation

### Minimal Create

```bash
curl -s -X POST -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?>
<methodCall>
  <methodName>wp.newPost</methodName>
  <params>
    <param><value><int>1</int></value></param>
    <param><value><string>admin</string></value></param>
    <param><value><string>APP_PASSWORD</string></value></param>
    <param><value><struct>
      <member><name>post_type</name><value><string>mb-views</string></value></member>
      <member><name>post_title</name><value><string>Team Members Grid</string></value></member>
      <member><name>post_content</name><value><string>TWIG_CODE_HERE</string></value></member>
      <member><name>post_status</name><value><string>publish</string></value></member>
    </struct></value></param>
  </params>
</methodCall>' \
  "https://example.com/xmlrpc.php"
```

### Get View by Slug: `/mbv/views` (GET-only, internal)

```bash
curl -s -u "admin:pass" \
  "https://example.com/wp-json/mbv/views"
```

Returns array of `mb-views` posts with all standard WP_Post fields. This is a **GET-only** hidden route — no write capability.

## TwigProxy Method Reference

The `mb` object in Twig is an instance of `MBViews\TwigProxy`. Its `__call` magic method proxies any unknown method to a global PHP function.

### Explicitly defined methods:

| Method | Source | Notes |
|--------|--------|-------|
| `get_posts(args)` | `new WP_Query(args)` | Returns array of Post renderer objects |
| `get_post(id)` | `get_post()` → Post renderer | Returns single Post renderer |
| `get_term(id, taxonomy)` | `get_post()` → Term renderer | ⚠️ Note: calls `get_post()` not `get_term()` (possible bug) |
| `get_user(id)` / `get_userdata(id)` | `get_userdata()` → User renderer | Aliases |
| `get_terms(args)` | `get_terms()` | Returns array of Term renderer objects |
| `get_users(args)` | `get_users()` | Returns array of User renderer objects |
| `map(field, width, height, zoom, ...)` | `rwmb_the_value()` | Map rendering |
| `checkbox(value, checked, unchecked)` | Ternary | Yes/no display |
| `post_comments()` | `comments_template()` | Renders comment template |

### Proxied via `__call`:

Any PHP function can be called as `mb.function_name()`. Examples:
- `mb.get_the_date()` → `get_the_date()`
- `mb.the_content()` → `the_content()`
- `mb.get_permalink(id)` → `get_permalink(id)`
- `mb.wp_get_attachment_image(id)` → `wp_get_attachment_image(id)`

## Twig Property Access Chain

```
{{ member.role }}
  → Post::__get('role')
    → Post::get_data() (lazy-loaded)
      → Renderer::get_single_value($post)  → WP post fields
      → Post::get_fields()
        → rwmb_get_registry('meta_box')->get_by(...)
        → filters meta_boxes for this post_type
        → meta_box_renderer->get_data(meta_box, 'post', post_id)
          → returns ALL field values for this post
    → $this->data['role'] ?? null
```

This means `{{ member.any_field_id }}` accesses any MetaBox field value directly, regardless of whether the field group is in Local JSON or the Builder DB.

## Source Files Referenced

- `/vendor/meta-box/mb-views/src/Renderer.php` — The main renderer
- `/vendor/meta-box/mb-views/src/TwigLoader.php` — Loads template from `post_content`
- `/vendor/meta-box/mb-views/src/TwigProxy.php` — `mb` object implementation
- `/vendor/meta-box/mb-views/src/Renderer/Post.php` — Post data access with `__get`
- `/vendor/meta-box/mb-views/src/Renderer/Base.php` — Base `__isset` + `__get` pattern
- `/vendor/meta-box/mb-views/src/PostType.php` — Post type registration (`supports: title, revisions`)
- `/vendor/meta-box/mb-views/src/Export.php` — Shows meta key names (`code`, `mode`, `type`, `singular_locations`, etc.)
- `/vendor/meta-box/mb-views/src/Import.php` — Import JSON flow (`wp_insert_post` + `update_post_meta` for settings)