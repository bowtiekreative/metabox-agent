# Front Page Team Member Display — Client-Side Rendering Pattern

A proved working approach for displaying CPT data on a static front page when MB Views is unavailable or REST API access is limited.

## Overview

- Create a static WordPress page with embedded HTML/CSS/JS
- Set it as the site's front page via `/wp/v2/settings`
- JavaScript fetches CPT data from REST API **client-side** on page load
- Skills from taxonomy are mapped from IDs to names via a second API call

## Step-by-Step

### 1. Create the page content

Embed a `<!-- wp:html -->` block in the page content containing:

```html
<style>
/* Card layout — grid, hover effects, avatar, skills tags, social links */
.team-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(340px, 1fr)); gap: 32px; }
.team-card { background: #fff; border-radius: 16px; padding: 32px; box-shadow: 0 4px 24px rgba(0,0,0,0.08); }
/* ... full styling as generated in session */
</style>

<div class="team-section">
  <h1>Our Team</h1>
  <p class="subtitle">Meet the talented people behind the project</p>
  <div id="team-grid" class="team-grid">
    <div style="text-align:center;color:#999;padding:40px;">Loading team members...</div>
  </div>
</div>

<script>
document.addEventListener('DOMContentLoaded', async function() {
  try {
    const [membersRes, skillsRes] = await Promise.all([
      fetch('/wp-json/wp/v2/team_member?_embed'),
      fetch('/wp-json/wp/v2/skill?per_page=100')
    ]);
    const members = await membersRes.json();
    const skillsList = await skillsRes.json();
    const skillMap = {};
    skillsList.forEach(s => { skillMap[s.id] = s.name; });

    const grid = document.getElementById('team-grid');
    if (!members.length) {
      grid.innerHTML = '<div>No team members found.</div>';
      return;
    }
    grid.innerHTML = members.map(m => {
      const initial = (m.title.rendered || '?')[0].toUpperCase();
      const skillNames = (m.skill || []).map(id => skillMap[id]).filter(Boolean);
      const skillsHtml = skillNames.length
        ? '<div class="skills">' + skillNames.map(s => `<span class="skill-tag">${s}</span>`).join('') + '</div>'
        : '';
      return `<div class="team-card">
        <div class="avatar">${initial}</div>
        <h3>${m.title.rendered}</h3>
        <div class="role">Team Member</div>
        <div class="bio">${m.content.rendered || 'No bio available.'}</div>
        ${skillsHtml}
        <div class="social-links">
          <a class="social-link" href="${m.link}">View Profile →</a>
        </div>
      </div>`;
    }).join('');
  } catch (err) {
    document.getElementById('team-grid').innerHTML = '<div>Could not load team members.</div>';
  }
});
</script>
```

### 2. Set the page as static front page

```bash
# Create or update the page
curl -X POST -u "admin:pass" \
  -d '{"title": "Home", "content": "<-- wp:html -->..."}' \
  "/wp-json/wp/v2/pages"

# Set as front page
curl -X PUT -u "admin:pass" \
  -d '{"show_on_front": "page", "page_on_front": <PAGE_ID>, "page_for_posts": <BLOG_PAGE_ID>}' \
  "/wp-json/wp/v2/settings"
```

### 3. Verify

Use browser or `curl ?nocache=$(date +%s)` — `web_extract` may return cached content from before the front page was set.

## Pitfalls

- **Skills taxonomy**: Team member `skill` field returns term **IDs**, not names. Always fetch `/wp/v2/skill?per_page=100` and build a mapping
- **REST URL paths**: Use relative paths (`/wp-json/wp/v2/...`) so the JS works regardless of domain
- **CPT must be REST-enabled**: The CPT must have `show_in_rest => true` (standard for MetaBox MB CPT extension)
- **Block themes**: Twenty Twenty-Five's `home.html` template can override front page settings — verify with cache busting