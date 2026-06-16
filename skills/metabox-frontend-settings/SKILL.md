---
name: metabox-frontend-settings
description: "Frontend forms with MB Frontend Submission, settings pages with MB Settings Page, user meta, and customizer integration"
version: 1.0.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [meta-box, frontend, forms, settings-pages, user-meta, customizer, wordpress, metabox]
    related_skills: [metabox-shortcodes, metabox-custom-fields]
---

# Frontend Forms, Settings Pages & User Meta — Skill Guide

Create front-end submission forms, settings pages (theme options), user meta fields, and customizer panels.

## MB Frontend Submission

### Adding a Frontend Form

**Shortcode:**
```php
[mb_frontend_form id="field-group-id" post_fields="title,content"]
```

**Gutenberg Block:** Insert "Submission Form" block.

### Key Shortcode Attributes

| Attribute | Description |
|-----------|-------------|
| `id` | Field group ID(s), comma-separated |
| `ajax` | Enable Ajax (`true`/`false`) |
| `edit` | Allow editing after submit (`true`/`false`) |
| `allow_delete` | Allow deleting submitted post |
| `post_type` | Submitted post type |
| `post_status` | Status for submitted posts |
| `post_fields` | Comma-separated: `title`, `content`, `excerpt`, `date`, `thumbnail` |
| `redirect` | Custom redirect URL after submission |
| `confirmation` | Success message text |
| `submit_button` | Submit button text |
| `recaptcha_key` / `recaptcha_secret` | Google reCaptcha v3 |

### Custom Table Model

```php
[mb_frontend_form id="field-group-id" object_type="model"]
```

### Hiding Fields from Frontend

In PHP field array:
```php
'hide_from_front' => true,
```

### Mixing Post Fields & Custom Fields

Remove `post_fields` from shortcode. Add post fields as custom fields with correct IDs:

| WordPress Field | Field ID |
|----------------|----------|
| Post title | `post_title` |
| Post content | `post_content` |
| Post excerpt | `post_excerpt` |
| Post date | `post_date` |
| Post thumbnail | `_thumbnail_id` |

**Example meta box with mixed fields:**
```php
$meta_boxes[] = [
    'title'  => 'Submit Bill',
    'id'     => 'bill_submit',
    'fields' => [
        ['name' => 'Submission Date', 'id' => 'submission_date', 'type' => 'date'],
        ['name' => 'Title', 'id' => 'post_title', 'type' => 'text'],
        ['name' => 'Type', 'id' => 'type', 'type' => 'select',
            'options' => ['docs' => 'Document', 'receipt' => 'Receipt']],
        ['name' => 'Description', 'id' => 'post_content', 'type' => 'textarea'],
        ['name' => 'Thumbnail', 'id' => '_thumbnail_id', 'type' => 'single_image'],
    ],
];
```

### Backend Validation

```php
add_filter('rwmb_frontend_validate', function ($validate, $config) {
    if ('bill_submit' !== $config['id']) return $validate;
    if (empty($_POST['submission_date'])) {
        return 'Submission date is required';
    }
    return $validate;
}, 10, 2);
```

### Dynamic Population

Populate fields from URL parameters:
```
?rwmb_frontend_field_object_id=123
```

Or via filter:
```php
add_filter('rwmb_frontend_field_value_post_id', function ($value, $args) {
    if ($args['id'] === 'your_meta_box_id') {
        $value = 123;
    }
    return $value;
}, 10, 2);
```

## MB Settings Page

### Register a Settings Page (Code)

```php
add_filter('mb_settings_pages', function ($settings_pages) {
    $settings_pages[] = [
        'id'          => 'theme_slug',
        'option_name' => 'theme_options',
        'menu_title'  => 'Theme Options',
        'parent'      => 'themes.php',
        'capability'  => 'edit_theme_options',
        'style'       => 'no-boxes',
        'columns'     => 2,
        'tabs'        => [
            'general' => 'General Settings',
            'design'  => [
                'label' => 'Design Customization',
                'icon'  => 'dashicons-admin-customizer',
            ],
        ],
        'tab_style'   => 'left',
        'customizer'  => true,
    ];
    return $settings_pages;
});
```

### Key Settings Page Parameters

| Parameter | Description |
|-----------|-------------|
| `id` | **Required.** Internal ID |
| `option_name` | DB option name (defaults to `id`) |
| `menu_title` | Menu text |
| `parent` | WordPress menu ID (e.g., `themes.php`) |
| `capability` | Default: `edit_theme_options` |
| `position` | Menu position number |
| `icon_url` | For top-level menus |
| `style` | `'boxes'` (meta-box style) or `'no-boxes'` |
| `columns` | 1 or 2 |
| `tabs` | Array of `key => label` or `key => ['label' => ..., 'icon' => ...]` |
| `tab_style` | `'default'` (top) or `'left'` |
| `customizer` | Show in Customizer |
| `network` | Multisite network-wide |

### Attach Fields to Settings Page

```php
add_filter('rwmb_meta_boxes', function ($meta_boxes) {
    $meta_boxes[] = [
        'id'             => 'general_settings',
        'title'          => 'General',
        'settings_pages' => 'theme_slug',
        'tab'            => 'general',  // Match tab key
        'fields'         => [
            ['name' => 'Site Logo', 'id' => 'site_logo', 'type' => 'file_input'],
            ['name' => 'Phone', 'id' => 'phone', 'type' => 'text'],
            ['name' => 'Address', 'id' => 'address', 'type' => 'textarea'],
        ],
    ];
    return $meta_boxes;
});
```

> **Important:** When using tabs, define `tab` for ALL meta boxes. Missing it hides meta boxes.

### Display Settings Field in Theme

```php
$phone = rwmb_meta('phone', ['object_type' => 'setting'], 'theme_options');
// Or echo:
rwmb_the_value('phone', ['object_type' => 'setting'], 'theme_options');
```

### Backup & Restore

Add a textarea with custom `type: "backup"`:

```php
[
    'name' => 'Backup / Restore',
    'id'   => 'settings_backup',
    'type' => 'textarea',
    'attributes' => ['type' => 'backup'],
],
```

## MB User Meta

### Register User Fields

```php
add_filter('rwmb_meta_boxes', function ($meta_boxes) {
    $meta_boxes[] = [
        'title'     => 'Profile Info',
        'type'      => 'user',  // Set to 'user' for user meta
        'fields'    => [
            ['name' => 'Phone', 'id' => 'user_phone', 'type' => 'tel'],
            ['name' => 'Position', 'id' => 'user_position', 'type' => 'text'],
            ['name' => 'Avatar', 'id' => 'user_avatar', 'type' => 'single_image'],
        ],
    ];
    return $meta_boxes;
});
```

### Display User Meta

```php
$phone = rwmb_meta('user_phone', ['object_type' => 'user'], $user_id);
```

### Frontend User Profile Form

```php
[mb_user_profile id="field-group-id"]
```

## Customizer Integration

Settings pages integrate with the Customizer. Each **settings page** becomes a Customizer **panel**, each **meta box** becomes a **section**.

Enable in code:
```php
'customizer' => true,
'customizer_only' => true,  // Only show in Customizer
```

### Top-Level Sections

Replace `settings_pages` with `panel => ''`:

```php
$meta_boxes[] = [
    'id'    => 'general',
    'title' => 'General',
    'panel' => '',
    'fields' => [...],
];
```