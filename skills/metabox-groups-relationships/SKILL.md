---
name: metabox-groups-relationships
description: "Cloneable groups, group fields, relationships, and conditional logic — complex data structures with MetaBox"
version: 1.0.0
author: MetaBox Agent
license: GPL-2.0+
metadata:
  hermes:
    tags: [meta-box, groups, cloning, relationships, conditional-logic, wordpress, metabox]
    related_skills: [metabox-custom-fields, metabox-views]
---

# Groups, Cloning & Relationships — Skill Guide

Create complex data structures: nested groups, clonable fields, bi-directional relationships, and conditional logic.

## Cloneable Fields

The `clone` feature makes **any field type** repeatable.

```php
[
    'name'  => 'Dates & Times',
    'id'    => 'dates',
    'type'  => 'datetime',
    'clone' => true,
]
```

### Clone Settings

| Setting | Key | Description |
|---------|-----|-------------|
| Cloneable | `clone` | `true`/`false`. Make field repeatable |
| Sortable | `sort_clone` | Enable drag-and-drop reordering |
| Clone default value | `clone_default` | Clone default value |
| Clone as multiple | `clone_as_multiple` | Store in separate DB rows (queryable) |
| Max clones | `max_clone` | Maximum number (must be > 2) |
| Min clones | `min_clone` | Minimum number |
| Add more text | `add_button` | Button text. Default: "+ Add more" |
| Clone empty start | `clone_empty_start` | Show only "Add more" initially |

### Default Values for Cloneable Fields

```php
'clone' => true,
'std' => [
    '2026-06-01',
    '2026-06-02',
],
```

### Querying Cloneable Fields

Serialized default: not queryable with `WP_Query` meta queries.

**Solution:** Set `'clone_as_multiple' => true` for queryable clones:

```php
$args = [
    'post_type'  => 'event',
    'meta_query' => [
        [
            'key'     => 'start_date',
            'value'   => ['2026-06-01', '2026-06-30'],
            'compare' => 'BETWEEN',
        ],
    ],
];
$query = new WP_Query($args);
```

## Group Fields

Nest fields inside a Group field for structured data.

```php
[
    'name'   => 'Speaker',
    'id'     => 'speaker',
    'type'   => 'group',
    'fields' => [
        [
            'name' => 'Name',
            'id'   => 'name',
            'type' => 'text',
        ],
        [
            'name' => 'Photo',
            'id'   => 'photo',
            'type' => 'single_image',
        ],
        [
            'name' => 'Bio',
            'id'   => 'bio',
            'type' => 'textarea',
        ],
        [
            'name' => 'Email',
            'id'   => 'email',
            'type' => 'email',
        ],
    ],
],
```

### Cloneable Groups

Make the group cloneable for repeatable entries:

```php
[
    'name'   => 'Speakers',
    'id'     => 'speakers',
    'type'   => 'group',
    'clone'  => true,
    'sort_clone' => true,
    'fields' => [
        [
            'name' => 'Name',
            'id'   => 'name',
            'type' => 'text',
        ],
        // ... sub-fields
    ],
],
```

### Displaying Group Values in PHP

```php
$speakers = rwmb_meta('speakers');
foreach ($speakers as $speaker) : ?>
    <div class="speaker">
        <img src="<?= $speaker['photo']['url'] ?>">
        <h3><?= $speaker['name'] ?></h3>
        <p><?= $speaker['bio'] ?></p>
    </div>
<?php endforeach;
```

### In Twig (MB Views)

```twig
{% for speaker in post.speakers %}
    <div class="speaker">
        <img src="{{ speaker.photo }}">
        <h3>{{ speaker.name }}</h3>
        <p>{{ speaker.bio }}</p>
    </div>
{% endfor %}
```

## MB Relationships

Creates **bi-directional relationships** between posts, terms, and users. Uses custom `mb_relationships` table.

### Supported Connection Types

- Post ↔ Post (including same-type)
- Post ↔ Term
- Post ↔ User
- Term ↔ Term
- User ↔ User
- Same-type (e.g., Event ↔ Event with `reciprocal: true`)

### Register Relationship (Code)

Always wrap in `mb_relationships_init`:

```php
add_action('mb_relationships_init', function () {
    MB_Relationships_API::register([
        'id'   => 'events_to_speakers',
        'from' => 'event',
        'to'   => 'speaker',
    ]);
});
```

### With Full Settings

```php
add_action('mb_relationships_init', function () {
    MB_Relationships_API::register([
        'id'   => 'events_to_speakers',
        'from' => [
            'object_type'   => 'post',
            'post_type'     => 'event',
            'empty_message' => 'No speakers selected',
            'meta_box'      => [
                'title'   => 'Speakers at this Event',
                'context' => 'side',
            ],
            'field' => [
                'max_items' => 10,
                'query_args' => [
                    'posts_per_page' => -1,
                ],
            ],
            'admin_column' => [
                'position' => 'after title',
                'title'    => 'Speakers',
                'link'     => 'edit',
            ],
        ],
        'to' => [
            'object_type'   => 'post',
            'post_type'     => 'speaker',
            'empty_message' => 'No events linked',
        ],
        'reciprocal' => false,
    ]);
});
```

### From Term (Category) to Post

```php
add_action('mb_relationships_init', function () {
    MB_Relationships_API::register([
        'id'   => 'categories_to_posts',
        'from' => [
            'object_type' => 'term',
            'taxonomy'    => 'category',
        ],
        'to'   => 'post',
    ]);
});
```

### From User to Post

```php
add_action('mb_relationships_init', function () {
    MB_Relationships_API::register([
        'id'   => 'users_to_posts',
        'from' => ['object_type' => 'user'],
        'to'   => 'post',
    ]);
});
```

### Getting Connected Items

**Quick API:**
```php
$connected = MB_Relationships_API::get_connected([
    'id'   => 'events_to_speakers',
    'from' => get_the_ID(),
]);
foreach ($connected as $p) {
    echo $p->post_title;
}
```

**With WP_Query:**
```php
$connected = new WP_Query([
    'relationship' => [
        'id'   => 'events_to_speakers',
        'from' => get_the_ID(),
    ],
    'nopaging' => true,
]);
```

Backward query: replace `'from'` with `'to'`.

### Reciprocal Relationships (Same Post Type)

```php
MB_Relationships_API::register([
    'id'         => 'related_events',
    'from'       => 'event',
    'to'         => 'event',
    'reciprocal' => true,  // Shows one meta box for both sides
]);
```

## Conditional Logic

Show/hide fields based on other field values.

```php
[
    'name' => 'Registration Type',
    'id'   => 'registration_type',
    'type' => 'select',
    'options' => [
        'free'  => 'Free',
        'paid'  => 'Paid',
    ],
],
[
    'name' => 'Ticket Price',
    'id'   => 'ticket_price',
    'type' => 'number',
    'visible' => ['registration_type', '=', 'paid'],
],
[
    'name' => 'Special Instructions',
    'id'   => 'instructions',
    'type' => 'textarea',
    'visible' => [
        ['registration_type', '!=', 'free'],
    ],
],
```

### Conditional Logic Operators

`=`, `!=`, `>`, `<`, `>=`, `<=`, `contains`, `not contains`, `between`

### Multiple Conditions

```php
'visible' => [
    ['registration_type', '=', 'paid'],
    ['ticket_price', '>', 100],
],
```