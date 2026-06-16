---
name: metabox-custom-fields
description: "Register & display MetaBox custom fields with PHP code — field types, settings, display functions, validation"
version: 1.0.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [meta-box, wordpress, custom-fields, php, metabox]
    related_skills: [metabox-views, metabox-shortcodes, metabox-cpt, metabox-groups-relationships]
---

# MetaBox Custom Fields — Skill Guide

Register and display custom fields using PHP code for themes/plugins. Code-based registration enables version control, reusability, and portability.

## Field Registration

Hook into the `rwmb_meta_boxes` filter. Each meta box is an array with `title`, `post_types`, and `fields`.

### Basic Example

```php
add_filter('rwmb_meta_boxes', function ($meta_boxes) {
    $meta_boxes[] = [
        'title'      => 'Event Details',
        'post_types' => 'event',
        'fields'     => [
            [
                'name' => 'Date & Time',
                'id'   => 'event_datetime',
                'type' => 'datetime',
            ],
            [
                'name' => 'Location',
                'id'   => 'event_location',
                'type' => 'text',
            ],
            [
                'name'          => 'Map',
                'id'            => 'event_map',
                'type'          => 'osm',
                'address_field' => 'event_location',
            ],
        ],
    ];
    return $meta_boxes;
});
```

### Multiple Post Types

```php
'post_types' => ['event', 'venue'],
// Or string: 'post'
```

### Meta Box Settings

| Key | Description |
|-----|-------------|
| `id` | Unique ID (auto-generated from title if absent) |
| `title` | **Required** — group title |
| `post_types` | Post type(s) (string or array, lowercase). Default: `post` |
| `context` | Position on edit screen (see below) |
| `style` | `'default'` (boxed) or `'seamless'` (no wrapper) |
| `closed` | Collapsed by default? Default `false` |
| `priority` | `'high'` or `'low'`. Default `'high'` |
| `default_hidden` | Hidden by default in Screen Options |
| `autosave` | Auto-save fields? Default `false` |
| `class` | Custom CSS class for wrapper |

### Context Positions

| Value | Location |
|-------|----------|
| `normal` | Below post editor (default) |
| `advanced` | Below `normal` |
| `side` | Right sidebar |
| `form_top` | Top of post form (before title) |
| `after_title` | After post title |
| `after_editor` | After content editor (before `normal`) |
| `before_permalink` | Before permalink |

> **Gutenberg note:** Only `normal` and `side` contexts supported.

## Field Types (40+)

### Basic (WordPress-native)

| Type | Key | Description |
|------|-----|-------------|
| Checkbox | `checkbox` | Yes/No toggle |
| Checkbox list | `checkbox_list` | Multiple choice checkboxes |
| Radio | `radio` | Single choice |
| Select | `select` | Dropdown (single/multiple) |
| Text | `text` | Single-line input |
| Textarea | `textarea` | Multi-line input |

### Advanced (require JS libraries)

`background`, `button`, `button_group`, `color`, `custom_html`, `date`, `datetime`, `hidden`, `image_select`, `key_value`, `map` (Google Maps), `oembed`, `osm` (OpenStreetMap), `password`, `select_advanced` (Select2), `slider`, `switch`, `time`, `wysiwyg`

### HTML5 (browser-native)

`email`, `number`, `range`, `url`

### WordPress Object Selection

`post` (select posts), `sidebar`, `taxonomy` (sets terms, doesn't save meta), `taxonomy_advanced` (saves term IDs as meta), `user`

### Upload / Media

`file`, `file_advanced`, `file_input`, `file_upload`, `image`, `image_advanced`, `image_upload`, `single_image`, `video`

### Group

`group` — nested fields (see `metabox-groups-relationships`)

## Common Field Settings

| Key | Description |
|-----|-------------|
| `name` | Field label. Optional. Empty = 100% width |
| `id` | **Required.** Field ID. Used as `meta_key` in DB. Letters, numbers, underscores only |
| `type` | **Required.** Field type key |
| `label_description` | Text below label |
| `desc` | Description below field input |
| `std` | Default value |
| `placeholder` | Placeholder text |
| `required` | Boolean. Default `false` |
| `disabled` | Boolean. Default `false` |
| `readonly` | Boolean. Default `false` |
| `multiple` | Allow multiple values (e.g., select) |
| `clone` | Cloneable/repeatable (see groups skill) |
| `class` | Custom CSS class |
| `sanitize_callback` | Custom sanitization. Set `'none'` to bypass |
| `attributes` | Custom HTML5 attributes (e.g., `['maxlength' => 100]`) |
| `validation` | jQuery validation rules |
| `before` / `after` | Custom HTML before/after field |

### Options for Choice Fields

```php
'options' => [
    'us' => 'United States',
    'uk' => 'United Kingdom',
],
```

## Displaying Fields

### `rwmb_meta()` — Smart Helper

Returns HTML for oEmbed/Maps, raw value for everything else.

```php
rwmb_meta($field_id, $args, $object_id);
```

**Examples:**

```php
// Simple text
<p><?= rwmb_meta('event_location') ?></p>

// Conditional logic
$layout = rwmb_meta('layout', '', 123);
if ($layout === 'content-sidebar') { /* ... */ }

// Single image (full size)
$image = rwmb_meta('cover', ['size' => 'full']);
<img src="<?= $image['url'] ?>">

// Image gallery with lightbox
$images = rwmb_meta('gallery', ['size' => 'thumbnail']);
foreach ($images as $image) : ?>
  <a href="<?= $image['full_url'] ?>" class="lightbox">
    <img src="<?= $image['url'] ?>">
  </a>
<?php endforeach;

// Term meta
$color = rwmb_meta('color', ['object_type' => 'term'], 15);

// Settings page
$hotline = rwmb_meta('hotline', ['object_type' => 'setting'], 'site_option');

// Custom table
$price = rwmb_meta('price', ['storage_type' => 'custom_table', 'table' => 'properties'], 15);
```

### `rwmb_get_value()` — Raw Value

Returns the stored value (not label/value for choice fields).

```php
$value = rwmb_get_value($field_id, $args, $object_id);
```

### `rwmb_the_value()` — HTML Output

Outputs human-readable HTML. For choice fields, returns **labels**, not values.

```php
rwmb_the_value($field_id, $args, $object_id, $echo = true);
```

## Validation

### Basic HTML5

```php
[
    'type'     => 'text',
    'id'       => 'phone',
    'required' => true,
    'pattern'  => '[0-9]{9}',
    'attributes' => ['minlength' => 9],
]
```

### jQuery Validation Plugin

```php
'validation' => [
    'rules' => [
        'email' => [
            'required'  => true,
            'minlength' => 7,
        ],
    ],
    'messages' => [
        'email' => [
            'required'  => 'Email is required',
            'minlength' => 'Email must be at least 7 characters',
        ],
    ],
],
```

## Fallback Functions

Always provide for production sites:

```php
if (!function_exists('rwmb_meta')) {
    function rwmb_meta($key, $args = [], $object_id = null) {
        return null;
    }
}
if (!function_exists('rwmb_the_value')) {
    function rwmb_the_value($key, $args = [], $object_id = null) {
        return null;
    }
}
```