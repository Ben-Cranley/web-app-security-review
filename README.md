# Web App Security Review

A security review of a deliberately vulnerable Flask web app. We attacked the site, found 10 vulnerabilities from the OWASP Top 10, then fixed each one in the code and re-tested it.

Third-year pair project, Secure Programming, TU Dublin (2025). Ben Cranley and Luke Douglas.

The vulnerable app was provided for the module, so this repository holds the write-up and our fixes rather than the full site.

## Summary

| # | Vulnerability | OWASP category | How we exploited it | Fix | Done by |
|---|---|---|---|---|---|
| 1 | SQL injection | A03 Injection | An always-true `OR` condition and a SQL comment on the login form, to log in without a password | Parameterised query | Ben |
| 2 | Stored XSS | A03 Injection | An image tag in a comment sent visitors' cookies to a server I ran | Removed the `\|safe` filter so Jinja escapes output | Ben |
| 3 | Reflected XSS | A03 Injection | HTML and CSS injected through the search query in the URL | Removed the `\|safe` filter | Ben |
| 4 | Path traversal | A01 Broken Access Control | `../` in the download link to download the whole database | Resolve the path and check it stays inside the docs folder | Ben |
| 5 | Sensitive data exposure | A02 Cryptographic Failures | Password typed in plain text, and full card numbers on the profile page | Password input type, and only the last 4 card digits shown | Ben |
| 6 | Broken access control | A01 Broken Access Control | Admin panel open to anyone who typed its URL | Session check before the page loads | Luke |
| 7 | IDOR | A01 Broken Access Control | Changed the user ID in the profile URL to view other users' bank details | Compare the session user with the requested profile | Luke |
| 8 | Packet sniffing | A02 Cryptographic Failures | Read a username and password from the login POST in Wireshark | Ran the site over HTTPS with a self-signed certificate | Luke |
| 9 | Open redirect | A01 Broken Access Control | Built a link on the real site that redirected to an outside page, usable for phishing | Only allow local paths starting with `/` and not `//` | Luke |
| 10 | Logging failures | A09 Logging and Monitoring Failures | None of the attacks above left any record | Warning-level logging for failed logins, blocked redirects, traversal and IDOR attempts | Luke |

Tools used: Burp Suite, Wireshark, OpenSSL and a browser.

## My fixes

### 1. SQL injection

The login query built the SQL with an f-string, so whatever the user typed became part of the query.

```python
# Before
query = text(f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'")
user = db.session.execute(query).fetchone()
```

```python
# After: input is sent separately and never treated as SQL
sql = text("SELECT * FROM users WHERE username = :username AND password = :password")
user = connection.execute(sql, {"username": username, "password": password}).fetchone()
```

### 2 and 3. Stored and reflected XSS

Both templates marked user input as `|safe`, which turns off Jinja's automatic escaping.

```html
<!-- Before -->
<li><strong>{{ comment.username }}</strong>: {{ comment.text|safe }}</li>
<h3>Search results for: {{ query|safe }}</h3>

<!-- After -->
<li><strong>{{ comment.username }}</strong>: {{ comment.text }}</li>
<h3>Search results for: {{ query }}</h3>
```

Scripts now show as plain text instead of running.

### 4. Path traversal

```python
base_directory = os.path.join(os.path.dirname(__file__), 'docs')
file_path = os.path.abspath(os.path.join(base_directory, file_name))

if not file_path.startswith(base_directory):
    logging.warning(f"path traversal attempt to: {file_name}")
    return "Unauthorized access attempt!", 403
```

### 5. Sensitive data exposure

```html
<input type="password" id="password" name="password" required>

<p><strong>Card Number:</strong> **** **** **** {{ card.cardno[-4:] }}</p>
```

## What I'd improve next

Looking back at the fixed app, these are the next things I'd change:

- Store passwords as salted hashes (for example bcrypt) instead of plain text.
- Stop storing and showing the CVV at all, as PCI DSS doesn't allow it.
- Use parameters in the profile and card queries too, even though Flask's `int` route converter already blocks injection there.
- Move the secret key and certificate paths into environment variables, and turn off debug mode.
- Check that a user is an admin, not just logged in, before showing the admin panel.
