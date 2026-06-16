---
name: metabox-views
description: "MB Views — Twig-based frontend templates with MetaBox fields, location rules, and PHP integration"
version: 1.0.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [meta-box, views, twig, templates, wordpress, metabox]
    related_skills: [metabox-custom-fields, metabox-shortcodes]
---

# MetaBox Views — Skill Guide

MB Views enables **Twig-based front-end templates** using Meta Box fields, post fields, site settings, user fields, and query fields. Customize any WordPress template.

## Creating a View

1. Navigate to **Meta Box > Views → Add New**
2. Two main areas:
   - **Template editors**: HTML/shortcodes + CSS + JavaScript
   - **Settings**: Location rules for the view

## Inserting Fields

Click **Insert Field** to open a panel with 4 tabs:

| Tab | Content |
|-----|---------|
| **Post** | Post fields + custom fields for posts |
| **Site** | Site fields + settings fields (from MB Settings Page) |
| **User** | User fields + custom fields (from MB User Meta) |
| **Query** | Loops and pagination for archive pages |

- Fields with options → popup for configuration
- Fields without options → snippet inserted immediately

### Cloneable Fields (repeat icon)

Generates a `for` loop:

```twig
{% for clone in post.tickets %}
    Content varies by field type
{% endfor %}
```

### Group Fields (arrow icon)

Click to toggle sub-fields. For cloneable groups, insert `for` loop by clicking group title.

### Relationship Fields

Found in **Query** tab. Shows "from" and "to" sides. Clicking inserts a loop of connected items.

## Including Other Views

Go to **Query** tab → click the view to include. Included views share parent's context. Enables composition.

## Running PHP Functions

Use the `mb` proxy as a transformer:

```twig
{% set post = mb.get_post(123) %}
{{ post.post_title }}
```

Normal PHP functions prefixed with `mb.`:

| PHP | Twig |
|-----|------|
| `get_post(123)` | `mb.get_post(123)` |
| `get_posts($args)` | `mb.get_posts(args)` |
| `rwmb_meta('field')` | `mb.rwmb_meta('field')` |
| `get_the_post_thumbnail(123)` | `mb.get_the_post_thumbnail(123)` |
| `custom_function()` | `mb.custom_function()` |

### Custom Query Example

```twig
{% set args = { post_type: 'event', posts_per_page: -1 } %}
{% set posts = mb.get_posts(args) %}
{% for post in posts %}
    <h3>{{ post.post_title }}</h3>
    {% set image = mb.rwmb_meta('event_image', '', post.ID) %}
    <img src="{{ image['full_url'] }}">
{% endfor %}
```

> **Important:** `get_posts()` returns `WP_Post` objects with different properties than Insert Field items.

### Complex Variables

```twig
{% set my_var = ["first", "second"] %}
{% set my_var = {first: "first value", second: "second value"} %}
{% set value = mb.custom_function(my_var) %}
```

## Custom Data

Pass custom data via shortcode attributes:

```php
[mbv id="your-view-id" name="Brian" age="50"]
```

Use in template:

```twig
<p><strong>Name:</strong> {{ name }}</p>
<p><strong>Age:</strong> {{ age }}</p>
```

## Locations

Configured in **Settings** meta box below the editor.

### Type Options

| Type | Description |
|------|-------------|
| **Singular** | Single pages |
| **Archive** | Archive pages |
| **Action** | Displays when an action fires |
| **Code** | PHP/WordPress conditional tags |
| **Shortcode** | Insert via `[mbv]` shortcode (available after saving) |
| **Block** | MB Blocks using `render: "view:view-name"` |

### Location Rules Logic

- **Groups**: Combined with `AND` (all groups must match)
- **Rules within groups**: Combined with `OR` (any rule can match)

### Render Type

1. **Whole page layout** (includes header/footer)
2. **Layout between header and footer** (full control)
3. **Post content area only** (theme controls layout)

### Position (for post content area)

`before`, `after`, or `replace` the content.

## Hooks

### `mbv_location_validate`

Custom location validation:

```php
add_filter('mbv_location_validate', function($result, $view, $type) {
    if ($view->post_name !== 'my-view') return $result;
    if (isset($_GET['custom_var']) && $_GET['custom_var'] == 1) return false;
    return $result;
}, 10, 3);
```

### `mbv_data`

Add custom data to Twig:

```php
add_filter('mbv_data', function($data, $twig) {
    $data['custom_var1'] = 'Value 1';
    return $data;
}, 10, 2);
```

## Twig Syntax Reference

### Tags

```twig
{% if condition %} ... {% endif %}
{% for item in items %} ... {% endfor %}
{% set var = value %}
```

### Output

```twig
{{ variable }}
{{ post.title }}
{{ post.thumbnail.full.url }}
```

### Filters

```twig
{{ value|default('fallback') }}
{{ post.content|excerpt(30) }}
{{ 'raw'|esc_html }}
```

## Common Patterns

### Single Post with Fields

```twig
<article class="event">
    <h1>{{ post.post_title }}</h1>
    <p class="date">{{ post.event_datetime }}</p>
    <div class="location">{{ post.event_location }}</div>
    {{ post.event_map }}
</article>
```

### Archive Loop with Pagination

```twig
{% for post in posts %}
    <div class="event-card">
        <h2>{{ post.post_title }}</h2>
        <p>{{ post.event_datetime }}</p>
    </div>
{% endfor %}
{{ posts.pagination }}
```

### Cloneable Group Loop

```twig
{% for speaker in post.speakers %}
    <div class="speaker">
        <h3>{{ speaker.name }}</h3>
        <p>{{ speaker.bio }}</p>
    </div>
{% endfor %}
```