---
name: metabox-custom-fields
description: "Register & display MetaBox custom fields — Local JSON, REST API behavior, data persistence, XML-RPC workarounds, field types, display functions, validation"
version: 1.2.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [meta-box, wordpress, custom-fields, php, metabox, rest-api, xml-rpc, local-json]
    related_skills: [metabox-views, metabox-shortcodes, metabox-cpt, metabox-groups-relationships, wordpress-code]
---

# MetaBox Custom Fields — Skill Guide

## Local JSON Field Registration

Fields can be registered via **Local JSON files** in the theme's `mb-json/` directory. These auto-load without needing to store field groups in the database.

### JSON Format
```json
{
  "$schema": "https://schemas.metabox.io/field-group.json",
  "id": "member_details",
  "title": "Member Details",
  "post_types": "team_member",
  "context": "normal",
  "style": "default",
  "fields": [
    {
      "name": "Role / Position",
      "id": "role",
      "type": "text",
      "required": true
    },
    {
      "name": "Short Bio",
      "id": "bio",
      "type": "textarea"
    },
    {
      "name": "Social Links",
      "id": "social_links",
      "type": "group",
      "clone": true,
      "fields": [
        {
          "name": "Platform",
          "id": "platform",
          "type": "select",
          "options": {
            "twitter": "Twitter / X",
            "linkedin": "LinkedIn",
            "github": "GitHub",
            "email": "Email",
            "website": "Website"
          }
        },
        {
          "name": "URL",
          "id": "url",
          "type": "url",
          "required": true
        }
      ]
    }
  ]
}
```

## Importing Local JSON to MetaBox Builder Database

When field groups are defined in Local JSON files (`mb-json/`), they render correctly on post edit screens but **do not appear in the MetaBox Builder admin UI** (Meta Box → Custom Fields). To make them visible and editable in the dashboard, import them to the Builder database.

### The `mbb/set-json-data` REST Endpoint

MetaBox Builder registers this endpoint at `/wp-json/mbb/set-json-data`:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `id` | ✅ Yes | The field group ID (matches `id` in the JSON file, e.g. `member_details`) |
| `use` | ✅ Yes | Method to call on `MBB\LocalJson`. Value is appended to `use_` to form method name |
| `post_type` | No | Defaults to `meta-box` |

Valid `use` values:
- `"json"` → calls `LocalJson::use_json()` — **sync from JSON file to database**
- `"database"` → calls `LocalJson::use_database()` — sync from database to JSON file

### How it Works

The endpoint calls `call_user_func([LocalJson::class, "use_$target"], ['post_name' => $id, 'post_type' => $post_type])`.

`use_json()`:
1. Finds the JSON file by `post_name` via `JsonService::get_json()`
2. Reads the file and parses it
3. Calls `sync_json()` which runs `wp_insert_post()` to create/update a `meta-box` CPT entry
4. Saves the field data as post meta on the new CPT entry
5. Writes the updated data back to the JSON file (via `use_database()`)

### Prerequisites for Sync

The `mb-json/` directory must be **writable by the web server user** (`www-data`). The `JsonService::get_paths()` method filters out unwritable paths:

```php
$paths = array_filter( $paths, function ( $path ) {
    return is_writable( $path );
} );
```

Fix permissions if needed:
```bash
chown -R www-data:www-data wp-content/themes/your-theme/mb-json/
chmod -R 755 wp-content/themes/your-theme/mb-json/
```

### Usage

```bash
# Import member_details field group from JSON to Builder DB
curl -X POST -u "admin:app_password" \
  -H "Content-Type: application/json" \
  -d '{"id": "member_details", "use": "json"}' \
  "https://example.com/wp-json/mbb/set-json-data"
# → {"success": true}
```

### Verification

Check which field groups are now in the Builder database:
```bash
curl -s -u "admin:pass" \
  "https://example.com/wp-json/mbb/json-data" | jq '.[].id'
# → "member_details"
# → "listing_details"
```

### The `mbb/json-data` Endpoint

Returns all field group configurations in the Builder database:
```bash
curl -s -u "admin:pass" \
  "https://example.com/wp-json/mbb/json-data"
# → [{"id": "member_details", ...}, {"id": "listing_details", ...}]
```

### When to Use Local JSON vs Builder DB

| Approach | Pros | Cons |
|----------|------|------|
| **Local JSON** | Version-controllable, portable, no DB dependency | Not editable in admin UI, REST API `meta_box` field stays empty |
| **Builder DB** | Editable in admin UI, REST API `meta_box` populates (for fields created in Builder, not imported from JSON) | Tied to database, harder to version-control |

**Recommended workflow**: Develop fields as Local JSON files, then import to Builder DB for production admin editing. Keep JSON files as a backup/source of truth.

### Saving field data via REST API — DOES NOT WORK for Local JSON
```bash
# This FAILS with "field does not exist" error:
curl -X POST -u "user:pass" \
  -d '{"meta_box": {"role": "Lead Developer"}}' \
  "/wp/v2/team_member/9"
# → {"code":"field_not_exists","message":"Field 'role' does not exists.","data":{"status":400}}
```

### XML-RPC Workaround for Data Persistence
When the REST API `meta_box` field rejects writes, use XML-RPC `wp.editPost` with `custom_fields`:

```bash
curl -X POST -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?>
<methodCall>
  <methodName>wp.editPost</methodName>
  <params>
    <param><value><int>1</int></value></param>
    <param><value><string>admin</string></value></param>
    <param><value><string>APP_PASSWORD</string></value></param>
    <param><value><int>POST_ID</int></value></param>
    <param><value><struct>
      <member>
        <name>custom_fields</name>
        <value><array><data>
          <value><struct>
            <member><name>key</name><value><string>role</string></value></member>
            <member><name>value</name><value><string>Lead WordPress Developer</string></value></member>
          </struct></value>
          <value><struct>
            <member><name>key</name><value><string>bio</string></value></member>
            <member><name>value</name><value><string>Bio text here</string></value></member>
          </struct></value>
        </data></array></value>
      </member>
    </struct></value></param>
  </params>
</methodCall>' \
  "https://example.com/xmlrpc.php"
```

**Note**: XML-RPC stores data as raw WordPress post meta. The data exists and is read by MetaBox display functions, but the REST API `meta_box` field still shows empty.

### Cloneable Groups in XML-RPC
For cloneable group fields (like `social_links`), the value must be PHP-serialized:

```
social_links => a:2:{i:0;a:2:{s:8:"platform";s:6:"github";s:3:"url";s:28:"https://github.com/johnsmith";}i:1;a:2:{s:8:"platform";s:7:"website";s:3:"url";s:22:"https://johnsmith.io";}}
```

### Verifying Saved Data
Use `metaWeblog.getPost` via XML-RPC and parse the `custom_fields` section:

```bash
php -r '
$xml = file_get_contents("https://example.com/xmlrpc.php", false, stream_context_create(["http" => ["method" => "POST", "header" => "Content-Type: text/xml\r\n", "content" => "..."]]));
$doc = new SimpleXMLElement($xml);
$vals = $doc->xpath("//member[name=\"custom_fields\"]//value//struct");
foreach ($vals as $v) {
    $members = $v->children();
    $kv = [];
    foreach ($members as $m) {
        $n = (string)$m->name;
        $vl = (string)$m->value->string ?? (string)$m->value->int ?? "";
        $kv[$n] = $vl;
    }
    if (isset($kv["key"]) && isset($kv["value"])) {
        echo $kv["key"] . " => " . $kv["value"] . "\n";
    }
}
'
```

## Field Types Reference

| Type | Meta Key Storage | REST API `meta_box` | XML-RPC Compatible |
|------|------------------|---------------------|--------------------|
| text | `meta_key = field_id` | Only when DB-registered | ✅ |
| textarea | `meta_key = field_id` | Only when DB-registered | ✅ |
| single_image | `meta_key = field_id` (stores attachment ID) | Only when DB-registered | ✅ |
| group (cloneable) | `meta_key = field_id` (serialized PHP array) | Only when DB-registered | ✅ (serialized) |

## Reference Files
- `references/xml-rpc-payloads.md` — Proven working XML-RPC payload templates for saving MetaBox field data (text, textarea, cloneable groups with PHP serialization), verification scripts, and what doesn't work
## Pitfalls

- **"field does not exist" error**: Means MetaBox REST API can't validate Local JSON fields — they're not registered in the database
- **Empty `meta_box`**: Local JSON fields never populate this REST field, even with data saved via other means
- **Serialization for groups**: Cloneable groups must be PHP-serialized (`serialize()`) in XML-RPC — JSON won't work
- **Field `id` required in JSON**: Local JSON files must have an `id` field at the group level. Missing `id` produces warnings in MetaBox Builder admin
- **Permissions**: CPT plugin files must be readable by www-data (`chmod 755`) or they won't load, causing CPTs to appear absent
- **`mb-json/` must be writable**: `JsonService::get_paths()` silently filters out unwritable directories. If `mb-json/` is owned by root, the Builder sync (`mbb/set-json-data`) and JSON auto-loading both fail silently. Always `chown -R www-data:www-data mb-json/`
- **`mbb/set-json-data` error handling**: A 500 error with `call_user_func(): Argument #1 ($callback) must be a valid callback, class MBB\LocalJson does not have a method "use_X"` means you passed an invalid `use` value. Valid values: `"json"` (JSON→DB), `"database"` (DB→JSON)
- **XML-RPC `wp.editPost` doesn't return custom_fields**: Use `metaWeblog.getPost` instead to verify saved data — `wp.getPost` omits the `custom_fields` section
- **REST API caches after static front page change**: After setting `page_on_front` via `/wp/v2/settings`, `web_extract` may return stale blog index content. Use `?nocache=` query param or `browser_vision` (screenshot) to verify live rendering