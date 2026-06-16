---
name: wordpress-code
description: "WordPress code modification — adding hooks, filters, actions, theme/plugin development patterns, and MetaBox integration"
version: 1.0.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [wordpress, hooks, filters, actions, php, theme, plugin, metabox]
    related_skills: [metabox-custom-fields, wordpress-local]
---

# WordPress Code Modification — Skill Guide

Patterns and best practices for modifying WordPress code — adding hooks, filters, actions, and integrating MetaBox custom fields in themes and plugins.

## Where to Place Code

### Theme `functions.php`

Best for site-specific customizations:

```php
// your-theme/functions.php
add_filter('rwmb_meta_boxes', function ($meta_boxes) {
    // MetaBox field groups
    return $meta_boxes;
});
```

**Always provide fallback functions** for production sites:
```php
if (!function_exists('rwmb_meta')) {
    function rwmb_meta($key, $args = [], $object_id = null) { return null; }
}
```

### Custom Plugin (recommended for portability)

```php
<?php
/**
 * Plugin Name: Site Custom Fields
 * Description: MetaBox field groups for this site
 * Version: 1.0.0
 */

// Prevent direct access
defined('ABSPATH') || exit;

// MetaBox fields
add_filter('rwmb_meta_boxes', function ($meta_boxes) {
    // ... field definitions
    return $meta_boxes;
});

// Fallback
if (!function_exists('rwmb_meta')) {
    function rwmb_meta($key, $args = [], $object_id = null) { return null; }
}
```

## Common Hooks for MetaBox Integration

### Display Fields in Single Template

```php
// In single.php or single-event.php
add_action('the_content', function ($content) {
    if (!is_singular('event')) return $content;

    $fields = '<div class="event-meta">';
    $fields .= '<p><strong>Date:</strong> ' . rwmb_meta('event_date') . '</p>';
    $fields .= '<p><strong>Location:</strong> ' . rwmb_meta('event_location') . '</p>';
    $fields .= '</div>';

    return $content . $fields;
});
```

### Display Fields Before/After Content

```php
// Before content
add_action('the_content', function ($content) {
    if (!is_singular('event')) return $content;
    return rwmb_the_value('event_banner') . $content;
});

// After content
add_filter('the_content', function ($content) {
    if (!is_singular('event')) return $content;
    return $content . rwmb_meta('event_gallery_html', ['size' => 'thumbnail']);
});
```

### Archive Page Customization

```php
// In archive-event.php
while (have_posts()) : the_post();
    $date = rwmb_meta('event_date');
    $location = rwmb_meta('event_location');
    ?>
    <article class="event-card">
        <h2><a href="<?php the_permalink(); ?>"><?php the_title(); ?></a></h2>
        <p class="event-date"><?= date('F j, Y', strtotime($date)) ?></p>
        <p class="event-location">📍 <?= $location ?></p>
    </article>
<?php endwhile;
```

### Admin Column for Custom Fields

```php
// Add column in admin list
add_filter('manage_event_posts_columns', function ($columns) {
    $columns['event_date'] = 'Event Date';
    return $columns;
});

// Display column value
add_action('manage_event_posts_custom_column', function ($column, $post_id) {
    if ($column === 'event_date') {
        echo rwmb_meta('event_date', '', $post_id);
    }
}, 10, 2);

// Make sortable
add_filter('manage_edit-event_sortable_columns', function ($columns) {
    $columns['event_date'] = 'event_date';
    return $columns;
});
```

### Quick Edit / Bulk Edit

```php
// Display field in quick edit
add_action('quick_edit_custom_box', function ($column_name, $post_type) {
    if ($column_name !== 'event_date' || $post_type !== 'event') return;
    ?>
    <fieldset class="inline-edit-col-right">
        <div class="inline-edit-col">
            <label><span class="title">Event Date</span>
                <input type="date" name="event_date" value="">
            </label>
        </div>
    </fieldset>
    <?php
}, 10, 2);
```

## MetaBox Hooks Reference

### `rwmb_meta_boxes`
Filter to register meta boxes and fields.

### `rwmb_frontend_validate`
Backend validation for frontend forms.

### `mbv_location_validate`
Custom location validation for MB Views.

### `mbv_data`
Add custom data to Twig in MB Views.

### `rwmb_meta_shortcode_secure`
Bypass HTML filtering in `[rwmb_meta]` shortcode.

### `rwmb_frontend_field_value_{$attribute}`
Dynamically populate frontend form fields.

## Bundling MetaBox in Themes/Plugins

### TGM Plugin Activation (Recommended)

```php
require_once __DIR__ . '/class-tgm-plugin-activation.php';

add_action('tgmpa_register', 'register_required_plugins');

function register_required_plugins() {
    $plugins = [
        [
            'name'     => 'Meta Box',
            'slug'     => 'meta-box',
            'required' => true,
        ],
        [
            'name'     => 'MB Custom Post Type',
            'slug'     => 'mb-custom-post-type',
            'required' => false,
        ],
    ];
    tgmpa($plugins);
}
```

## Quick Reference

| Hook | Purpose |
|------|---------|
| `the_content` | Modify post content (add fields) |
| `manage_{post_type}_posts_columns` | Add admin columns |
| `manage_{post_type}_posts_custom_column` | Display admin column values |
| `quick_edit_custom_box` | Quick edit fields |
| `rwmb_meta_boxes` | Register MetaBox field groups |
| `rwmb_frontend_validate` | Validate frontend forms |
| `mbv_data` | Pass custom data to Twig views |