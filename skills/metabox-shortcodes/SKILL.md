---
name: metabox-shortcodes
description: "All MetaBox shortcodes — [rwmb_meta], [mb_frontend_form], [mb_frontend_post], [mb_user_profile], [mb_relationships], [mbv]"
version: 1.0.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [meta-box, shortcodes, wordpress, metabox]
    related_skills: [metabox-custom-fields, metabox-views, metabox-frontend-settings]
---

# MetaBox Shortcodes — Skill Guide

MetaBox provides shortcodes to display custom fields, forms, user profiles, and relationships anywhere in WordPress content or widgets.

## `[rwmb_meta]` — Display Custom Fields

Display a single field value in post content or widgets. Works like `rwmb_the_value()`.

```php
[rwmb_meta id="field_id" object_id="15"]
```

### Attributes

| Attribute | Description |
|-----------|-------------|
| `id` | **Required.** The field ID |
| `object_id` | Optional. Object ID (post, term, user). Defaults to current post |
| `attribute` | Get a single attribute from field value (e.g., URL of image, term slug) |
| `render_shortcode` | Render inner shortcodes inside field value. Default `true` |
| Other | Passed as 2nd parameter to `rwmb_the_value()` |

### Examples

**Basic text field:**
```php
[rwmb_meta id="phone"]
```

**Single image URL (thumbnail size):**
```php
[rwmb_meta id="company_logo" attribute="url" size="thumbnail"]
```

**Term meta:**
```php
[rwmb_meta id="color" object_id="15" object_type="term"]
```

**Settings page value:**
```php
[rwmb_meta id="phone" object_id="site_option" object_type="setting"]
```

**Custom table:**
```php
[rwmb_meta id="area" object_id="15" storage_type="custom_table" table="properties"]
```

### Bypass HTML Filtering

By default, shortcode output is filtered by `wp_kses_post()`. Bypass globally or per-field:

```php
// Global bypass
add_filter('rwmb_meta_shortcode_secure', '__return_false');

// Per-field bypass
add_filter('rwmb_meta_shortcode_secure_footer_script', '__return_false');
```

## `[mb_frontend_form]` — Frontend Submission Form

Display a front-end submission form for a field group.

```php
[mb_frontend_form id="field-group-id" post_fields="title,content"]
```

### Attributes

| Attribute | Description |
|-----------|-------------|
| `id` | Field group ID(s), comma-separated for multiple |
| `ajax` | Enable Ajax submission (`true`/`false`, default `false`) |
| `edit` | Allow editing after submit (`true`/`false`) |
| `allow_delete` | Allow deleting submitted post |
| `force_delete` | Permanently delete or move to Trash |
| `show_add_more` | Show "Add New" button after submit |
| `object_type` | `post` or `model` (for custom tables) |
| `object_id` | Update existing object; use `current` for current post ID |
| `post_type` | Submitted post type (default: first post type in meta box) |
| `post_status` | Status for submitted posts |
| `post_fields` | Comma-separated: `title`, `content`, `excerpt`, `date`, `thumbnail` |
| `label_title`, `label_content`, `label_excerpt`, `label_date`, `label_thumbnail` | Custom labels |
| `submit_button` | Submit button text |
| `add_button` | "Add New" button text |
| `delete_button` | Delete button text |
| `redirect` | Custom redirect URL after submission |
| `confirmation` | Success message text |
| `delete_confirmation` | Deletion confirmation text |
| `recaptcha_key` / `recaptcha_secret` | Google reCaptcha v3 keys |

**Custom table model:**
```php
[mb_frontend_form id="field-group-id" object_type="model"]
```

## `[mb_frontend_post]` — Quick Post Submission

Simpler form for basic post submission:

```php
[mb_frontend_post post_fields="title,content,thumbnail"]
```

## `[mb_user_profile]` — User Profile Form

Display user info/edit/registration forms:

```php
[mb_user_profile id="field-group-id"]
```

## `[mb_relationships]` — Display Relationship Connections

Show connected items:

```php
[mb_relationships id="relationship-id" from="123"]
```

## `[mbv]` — MB Views Shortcode

Render a saved View:

```php
[mbv id="your-view-id" name="Brian" age="50"]
```

- `id` is the view slug (available after saving the view)
- Extra attributes become custom data in Twig template (accessible as `{{ name }}`, `{{ age }}`)

## Quick Reference

| Shortcode | Purpose |
|-----------|---------|
| `[rwmb_meta]` | Display a single field value |
| `[mb_frontend_form]` | Full front-end submission form |
| `[mb_frontend_post]` | Quick post submission form |
| `[mb_user_profile]` | User profile/registration form |
| `[mb_relationships]` | Display connected items |
| `[mbv]` | Embed a saved MB View |