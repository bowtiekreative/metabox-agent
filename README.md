# MetaBox Agent

A **Hermes Agent** specialized in **MetaBox development for WordPress** — creating custom fields, Views (Twig templates), shortcodes, custom post types, taxonomies, relationships, settings pages, and front-end forms. Includes tools for local WordPress testing.

## Architecture

```
metabox-agent/
├── README.md                    # This file
├── AGENT-PROMPT.md              # Master system prompt for the Metabox agent
└── skills/
    ├── metabox-custom-fields/   # Register & display custom fields with PHP
    ├── metabox-views/           # MB Views (Twig) — templates, locations, PHP in Twig
    ├── metabox-shortcodes/      # All MetaBox shortcodes: [rwmb_meta], [mb_frontend_form], etc.
    ├── metabox-cpt/             # Custom post types & taxonomies
    ├── metabox-groups-relationships/  # Cloneable groups, group fields, relationships, conditional logic
    ├── metabox-frontend-settings/     # Front-end submission, settings pages, user meta
    ├── wordpress-local/         # Docker WordPress setup for local testing
    └── wordpress-code/          # WordPress code modification with hooks, filters, actions
```

## Skills Overview

| Skill | Purpose |
|-------|---------|
| **metabox-custom-fields** | Register fields with `rwmb_meta_boxes` filter; 40+ field types; field settings; display with `rwmb_meta()`, `rwmb_get_value()`, `rwmb_the_value()` |
| **metabox-views** | MB Views Twig templates; Insert Field (post/site/user/query tabs); location rules; PHP via `mb.*` proxy; custom data |
| **metabox-shortcodes** | `[rwmb_meta]`, `[mb_frontend_form]`, `[mb_frontend_post]`, `[mb_user_profile]`, `[mb_relationships]`, `[mbv]` |
| **metabox-cpt** | Register CPTs & taxonomies with code; `register_post_type()`; `register_taxonomy()`; MB CPT extension integration |
| **metabox-groups-relationships** | Cloneable fields/groups; group fields; MB Relationships API; conditional logic; storage modes |
| **metabox-frontend-settings** | MB Frontend Submission; MB Settings Pages; MB User Meta; customizer integration |
| **wordpress-local** | Docker WordPress + MySQL; wp-cli; plugin activation; testing workflow |
| **wordpress-code** | `functions.php` edits; hooks/filters/actions; theme/plugin development patterns |

## Deployment

Deployed as a Hermes Agent Docker container on the VPS behind Traefik reverse proxy.

## Repository

- **Owner:** `bowtiekreative`
- **Name:** `metabox-agent`
- **Visibility:** Private
