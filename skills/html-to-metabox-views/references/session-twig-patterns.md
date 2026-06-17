# Session-Proven MB Views Twig Patterns

From the session that created team member display on `wordpress-plac.srv620544.hstgr.cloud`.

## Working Twig Template (stored in `post_content`)

```twig
<div class="mb-team-grid">
{% for member in mb.get_posts({post_type: "team_member", posts_per_page: -1}) %}
  <div class="mb-team-card">
    <h3>{{ member.post_title }}</h3>
    <p class="role">{{ member.role }}</p>
    <div class="bio">{{ member.bio }}</div>
  </div>
{% else %}
  <p>No team members found.</p>
{% endfor %}
</div>
```

## Key Twig Facts (confirmed in this session)

- `{{ member.role }}` works because the `Post` renderer merges MetaBox field data via `__get()` magic method
- `mb.get_posts()` returns post objects with MetaBox field values merged in
- `{% else %}` on `{% for %}` handles the empty-results case in Twig
- String escaping: use double quotes inside single-quoted XML attribute context, or vice versa

## XML-RPC Payload: Creating the View

The Twig template goes in `post_content`. The XML must be well-formed — use XML entities for special chars (`{%` / `%}` and Twig string quotes).

```xml
<member>
  <name>post_content</name>
  <value><string>ESCAPED_TWIG_HTML</string></value>
</member>
```

## XML-RPC Payload: Setting mode=shortcode

```xml
<member>
  <name>custom_fields</name>
  <value><array><data>
    <value><struct>
      <member><name>key</name><value><string>mode</string></value></member>
      <member><name>value</name><value><string>shortcode</string></value></member>
    </struct></value>
    <value><struct>
      <member><name>key</name><value><string>type</string></value></member>
      <member><name>value</name><value><string>shortcode</string></value></member>
    </struct></value>
  </data></array></value>
</member>
```

## mbb/set-json-data REST Endpoint (Builder DB Import)

Used to sync Local JSON field groups into the MetaBox Builder database:

```
POST /wp-json/mbb/set-json-data
Body: {"id": "member_details", "use": "json"}
Response: {"success": true}
```

The `use` parameter value becomes `use_{value}` method call on `MBB\LocalJson`.