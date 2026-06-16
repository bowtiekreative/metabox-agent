---
name: metabox-cpt
description: "Custom Post Types and Taxonomies — register CPTs/taxonomies with PHP code using MB CPT extension or native WordPress functions"
version: 1.0.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [meta-box, cpt, custom-post-types, taxonomies, wordpress, metabox]
    related_skills: [metabox-custom-fields, metabox-groups-relationships]
---

# Custom Post Types & Taxonomies — Skill Guide

Create custom post types (CPTs) and custom taxonomies with MetaBox's MB CPT extension or native WordPress functions.

## Using MB Custom Post Types Extension (UI)

Go to **Meta Box > Post Types → New Post Type** for UI creation. But always prefer **code-based** registration for version control.

## Get PHP Code from UI

After configuring a CPT in the UI, click **Get PHP Code** to extract the registration code. Then paste in `functions.php` (or plugin) and deactivate the extension for better performance.

## Code-Based CPT Registration

```php
add_action('init', function () {
    register_post_type('event', [
        'labels' => [
            'name'          => __('Events'),
            'singular_name' => __('Event'),
            'add_new_item'  => __('Add New Event'),
            'edit_item'     => __('Edit Event'),
            'view_item'     => __('View Event'),
            'search_items'  => __('Search Events'),
            'not_found'     => __('No events found'),
        ],
        'public'       => true,
        'has_archive'  => true,
        'menu_icon'    => 'dashicons-calendar',
        'supports'     => ['title', 'editor', 'thumbnail', 'excerpt'],
        'rewrite'      => ['slug' => 'events'],
        'show_in_rest' => true,  // For block editor
    ]);
});
```

### Key CPT Arguments

| Argument | Description |
|----------|-------------|
| `labels` | Array of labels (auto-generate from name + singular_name) |
| `public` | Show in front-end and admin. Default `false` |
| `has_archive` | Enable archive pages. Default `false` |
| `menu_icon` | Dashicons class or custom URL |
| `supports` | Core features: `title`, `editor`, `thumbnail`, `excerpt`, `author`, `comments`, `revisions`, `page-attributes`, `custom-fields`, `post-formats` |
| `rewrite` | Permalink settings: `['slug' => 'events']` |
| `show_in_rest` | Required for block editor (Gutenberg) |
| `hierarchical` | Like Pages. Default `false` |
| `exclude_from_search` | Opposite of `public` |
| `capability_type` | `'post'`, `'page'`, or `['book', 'books']` for custom capabilities |
| `menu_position` | Position in admin menu (e.g., 5 for below Posts) |

## Code-Based Taxonomy Registration

```php
add_action('init', function () {
    register_taxonomy('event_category', 'event', [
        'labels' => [
            'name'          => __('Event Categories'),
            'singular_name' => __('Event Category'),
            'search_items'  => __('Search Categories'),
            'all_items'     => __('All Categories'),
            'edit_item'     => __('Edit Category'),
            'add_new_item'  => __('Add New Category'),
        ],
        'hierarchical'  => true,  // Like categories (false = like tags)
        'public'        => true,
        'rewrite'       => ['slug' => 'event-category'],
        'show_in_rest'  => true,  // For block editor
        'show_admin_column' => true,  // Show in post list table
    ]);
});
```

### Key Taxonomy Arguments

| Argument | Description |
|----------|-------------|
| `hierarchical` | `true` = categories, `false` = tags |
| `public` | Show in front-end and admin |
| `show_admin_column` | Show in post type admin list |
| `rewrite` | Permalink slug |
| `show_in_rest` | Required for block editor |
| `capabilities` | Custom capabilities: `manage_terms`, `edit_terms`, `delete_terms`, `assign_terms` |

## Attaching Meta Boxes to CPTs

```php
add_filter('rwmb_meta_boxes', function ($meta_boxes) {
    $meta_boxes[] = [
        'title'      => 'Event Details',
        'post_types' => 'event',  // Your CPT slug
        'fields'     => [
            [
                'name' => 'Date',
                'id'   => 'event_date',
                'type' => 'date',
            ],
            // ... more fields
        ],
    ];
    return $meta_boxes;
});
```

### Multiple Post Types

```php
'post_types' => ['event', 'venue'],
```

### Taxonomy Attached to Multiple Post Types

```php
register_taxonomy('event_category', ['event', 'venue'], [...]);
```

## WordPress Menu IDs (for submenu pages)

| Menu | ID | Position |
|------|----|----------|
| Dashboard | `index.php` | 2 |
| Posts | `edit.php` | 5 |
| Media | `upload.php` | 10 |
| Pages | `edit.php?post_type=page` | 20 |
| Appearance | `themes.php` | 60 |
| Plugins | `plugins.php` | 65 |
| Settings | `options-general.php` | 80 |

## Quick Patterns

### CPT + Taxonomy + Meta Box

```php
// 1. Register CPT
add_action('init', function () {
    register_post_type('portfolio', [
        'labels'      => ['name' => 'Portfolio', 'singular_name' => 'Project'],
        'public'      => true,
        'has_archive' => true,
        'supports'    => ['title', 'editor', 'thumbnail'],
        'menu_icon'   => 'dashicons-portfolio',
        'show_in_rest' => true,
    ]);

    register_taxonomy('portfolio_category', 'portfolio', [
        'labels' => ['name' => 'Categories', 'singular_name' => 'Category'],
        'hierarchical' => true,
        'show_in_rest' => true,
    ]);
});

// 2. Add meta box
add_filter('rwmb_meta_boxes', function ($meta_boxes) {
    $meta_boxes[] = [
        'title'      => 'Project Details',
        'post_types' => 'portfolio',
        'fields'     => [
            ['name' => 'Client', 'id' => 'client', 'type' => 'text'],
            ['name' => 'URL', 'id' => 'project_url', 'type' => 'url'],
            ['name' => 'Screenshots', 'id' => 'screenshots', 'type' => 'image_advanced'],
        ],
    ];
    return $meta_boxes;
});
```

### Using MB CPT Code (from Get PHP Code)

```php
add_action('mb_cpt_init', function () {
    register_post_type('portfolio', [...]);
    register_taxonomy('portfolio_category', 'portfolio', [...]);
});
```