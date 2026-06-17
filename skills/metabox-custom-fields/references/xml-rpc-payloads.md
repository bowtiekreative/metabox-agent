# XML-RPC Payloads for MetaBox Field Data Persistence

These are **proven working payloads** from a real session where MetaBox Local JSON fields rejected REST API writes.

## Context
- MetaBox AIO 3.7.2
- Fields registered via `mb-json/` Local JSON files
- WordPress REST API `meta_box` field returned empty `[]` and rejected writes with `"field does not exist"`
- App password authentication used for both REST API and XML-RPC

## Prerequisites
- The WordPress XML-RPC endpoint: `https://yoursite.com/xmlrpc.php`
- App password or main admin password
- Know the post ID

## Basic Field Save (text, textarea)

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>wp.editPost</methodName>
  <params>
    <param><value><int>1</int></value></param>
    <param><value><string>admin</string></value></param>
    <param><value><string>APP_PASSWORD_HERE</string></value></param>
    <param><value><int>9</int></value></param>
    <param><value><struct>
      <member>
        <name>custom_fields</name>
        <value><array><data>
          <value><struct>
            <member><name>key</name><value><string>role</string></value></member>
            <member><name>value</name><value><string>Lead WordPress Developer</string></value></member>
          </struct></value>
          <value><struct>
            <member><name>key</name><value><string>bio</string></value></member>
            <member><name>value</name><value><string>Jane has over 10 years of experience building WordPress solutions.</string></value></member>
          </struct></value>
        </data></array></value>
      </member>
    </struct></value></param>
  </params>
</methodCall>
```

## Cloneable Group Fields (PHP-serialized)

The key insight: cloneable group values must be **PHP-serialized arrays**, not JSON.

### Serialization format
For a group with sub-fields `platform` (select, string) and `url` (url, string):

```
a:2:{i:0;a:2:{s:8:"platform";s:7:"twitter";s:3:"url";s:27:"https://twitter.com/janedoe";}i:1;a:2:{s:8:"platform";s:8:"linkedin";s:3:"url";s:31:"https://linkedin.com/in/janedoe";}}
```

Where:
- `a:2:{...}` = array with 2 elements
- `i:0` = integer key 0
- `a:2:{...}` = sub-array with 2 key-value pairs
- `s:8:"platform"` = string of length 8 = "platform"
- `s:7:"twitter"` = string of length 7 = "twitter"

### Generation in PHP
```php
$social_links = [
  ['platform' => 'twitter', 'url' => 'https://twitter.com/janedoe'],
  ['platform' => 'linkedin', 'url' => 'https://linkedin.com/in/janedoe'],
  ['platform' => 'email', 'url' => 'mailto:jane@example.com'],
];
echo serialize($social_links);
// → a:3:{i:0;a:2:{s:8:"platform";s:7:"twitter";s:3:"url";s:27:"...";}i:1;a:2:{s:8:"platform";s:8:"linkedin";s:3:"url";s:31:"...";}i:2;a:2:{s:8:"platform";s:5:"email";s:3:"url";s:23:"mailto:jane@example.com";}}
```

### Full XML-RPC payload with group
```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>wp.editPost</methodName>
  <params>
    <param><value><int>1</int></value></param>
    <param><value><string>admin</string></value></param>
    <param><value><string>APP_PASSWORD_HERE</string></value></param>
    <param><value><int>10</int></value></param>
    <param><value><struct>
      <member>
        <name>custom_fields</name>
        <value><array><data>
          <value><struct>
            <member><name>key</name><value><string>role</string></value></member>
            <member><name>value</name><value><string>UX Designer &amp; Frontend Engineer</string></value></member>
          </struct></value>
          <value><struct>
            <member><name>key</name><value><string>social_links</string></value></member>
            <member><name>value</name><value><string>a:2:{i:0;a:2:{s:8:"platform";s:6:"github";s:3:"url";s:28:"https://github.com/johnsmith";}i:1;a:2:{s:8:"platform";s:7:"website";s:3:"url";s:22:"https://johnsmith.io";}}</string></value></member>
          </struct></value>
        </data></array></value>
      </member>
    </struct></value></param>
  </params>
</methodCall>
```

## Verifying Saved Data

```bash
curl -s \
  -X POST \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?><methodCall><methodName>metaWeblog.getPost</methodName><params><param><value><int>9</int></value></param><param><value><string>admin</string></value></param><param><value><string>APP_PASSWORD_HERE</string></value></param></params></methodCall>' \
  "https://yoursite.com/xmlrpc.php" | \
  php -r '
$xml = file_get_contents("php://stdin");
$doc = new SimpleXMLElement($xml);
$vals = $doc->xpath("//member[name=\"custom_fields\"]//value//struct");
foreach ($vals as $v) {
    $members = $v->children();
    $kv = [];
    foreach ($members as $m) {
        $n = (string)$m->name;
        $vl = (string)$m->value->string ?? (string)$m->value->int ?? "";
        $kv[$n] = $vl;
    }
    if (isset($kv["key"]) && isset($kv["value"])) {
        echo $kv["key"] . " => " . substr($kv["value"], 0, 80) . "\n";
    }
}
'
```

## What DOESN'T Work

### REST API meta_box field (for Local JSON)
```bash
curl -X POST -u "admin:pass" \
  -d '{"meta_box": {"role": "Lead Developer"}}' \
  "/wp-json/wp/v2/team_member/9"
# → {"code":"field_not_exists","message":"Field 'role' does not exists.","data":{"status":400}}
```

### REST API meta field (unregistered meta keys)
```bash
curl -X POST -u "admin:pass" \
  -d '{"meta": {"role": "Lead Developer"}}' \
  "/wp-json/wp/v2/team_member/9"
# Meta value silently ignored (no error, no save) because meta key isn't registered with show_in_rest
```