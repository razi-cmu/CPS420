# Week 8: Web Security, Testing, and E-Commerce Law

Over seven weeks we built a bookstore that speaks XML-RPC, REST, and JSON, remembers carts in sessions, caches slow answers, stores books in a database, and runs a page that talks to its own API. Along the way we kept a list of shortcuts: views marked `csrf_exempt`, a debug server, a secret key that came with the project, a POST anyone can send twice. None of them mattered on a laptop. All of them matter the day the site goes online.

This week has three parts. First, how web applications get attacked and how they defend themselves. Second, how to prove a defense works, with automated tests and secure protocols. Third, the laws and rules that apply the moment a site sells something to real people.

## Thinking Like an Attacker

A vulnerability is a weakness, a threat is someone or something that could exploit it, and risk is how likely that is and how much it would hurt. Security work is mostly about risk: you can't fix everything, so you fix what is most likely and most damaging first.

Three properties describe what we are protecting, often called the CIA triad:

| Property | Meaning | Bookstore example of losing it |
|---|---|---|
| Confidentiality | Only the right people can read data | Someone reads another customer's cart or address |
| Integrity | Only the right people can change data | Someone sets every price to $0.01 |
| Availability | The service works when needed | A flood of requests takes the store offline on a sale day |

Two habits sit under every defense in this lecture. The first is never trusting the client. Anything that arrives in a request, including form fields, JSON bodies, cookies, and headers, can be typed by an attacker, not just sent by our page. The second is defense in depth: no single layer is perfect, so we stack several (validation, output encoding, CSRF tokens, HTTPS, database constraints, monitoring) and expect each to catch what the others miss.

## The OWASP Top 10

The Open Worldwide Application Security Project (OWASP) publishes a ranked list of the most serious web application risks, built from real vulnerability data and practitioner surveys. The current edition is the 2025 list:

| Rank | Category | Where it shows up in our bookstore |
|---|---|---|
| A01 | Broken Access Control | Anyone can POST a new book to v1 or v2; there are no permissions at all |
| A02 | Security Misconfiguration | `DEBUG = True`, empty `ALLOWED_HOSTS`, the generated `SECRET_KEY` |
| A03 | Software Supply Chain Failures | We install Django, DRF, and `requests` without checking them for known vulnerabilities |
| A04 | Cryptographic Failures | Session and CSRF cookies travel over plain HTTP |
| A05 | Injection | The f-string raw SQL query from Week 6 |
| A06 | Insecure Design | Duplicate POSTs were possible by design until the database said no |
| A07 | Authentication Failures | Weak admin passwords, no login rate limiting |
| A08 | Software or Data Integrity Failures | Loading scripts from a CDN without checking they haven't been altered |
| A09 | Security Logging and Alerting Failures | Nothing records or alerts on a burst of failed logins |
| A10 | Mishandling of Exceptional Conditions | v1 answering a bad price with a 500 and, in debug mode, a full stack trace |

Two things stand out compared with older lists. Supply chain failures moved near the top because modern apps are mostly other people's code. And a brand-new category, mishandling exceptional conditions, recognizes that error paths are where systems often fail open or leak information. We have been looking at error paths since Week 4.

## Three Classic Attacks

### Injection

Injection happens when data is pasted into something that gets interpreted as code: SQL, a shell command, an HTML page. Week 6 showed the safe and unsafe ways to run raw SQL:

```python
Book.objects.raw(f"SELECT * FROM catalog_book WHERE title = '{title}'")    # unsafe
Book.objects.raw("SELECT * FROM catalog_book WHERE title = %s", [title])   # safe
```

In the first line, the database sees one string that mixes our query with the user's text, and a quote character in the title can end our string and start new SQL. In the second, the query and the value travel separately, so the value can never become SQL. The ORM always does the second for you; injection reappears when developers build query strings by hand.

### Cross-site scripting (XSS)

XSS is injection into a page. If user data is inserted into HTML without escaping, a title like `<img src=x onerror="alert('XSS')">` becomes a real tag, and its script runs in every visitor's browser with that visitor's cookies and session. When the payload is saved in the database and served to everyone, it is stored XSS, the most damaging kind.

Django templates escape variables automatically, turning `<` into `&lt;`, which is why nothing in Weeks 1 to 6 was at risk. Week 7 moved rendering into JavaScript, so that job became ours: `textContent` treats data as text, `innerHTML` treats it as markup. A Content Security Policy (CSP) header adds a second layer by telling the browser which scripts it may run at all.

### Cross-site request forgery (CSRF)

Browsers attach cookies to every request to a site, including requests triggered by some other site's page. If you are logged in to the bookstore and visit a malicious page, that page can submit a form to the bookstore and your session cookie rides along. The attacker can't read the response, but the action still happens.

Django's defense, which we have used since Week 1, is a secret token the attacker's page can't know: `{% csrf_token %}` in forms, or the `X-CSRFToken` header in AJAX. `SameSite` cookies, which we set on the visit counter in Week 5, add another layer by telling the browser not to send the cookie on most cross-site requests. CSRF only matters when a cookie carries authority, which is why DRF checks it only for logged-in sessions. Our `csrf_exempt` views were safe enough while they had no logged-in users to impersonate, and would become a real hole the day they did.



## Secure Protocols: TLS, HTTPS, IPSec, and SFTP

### From SSL to TLS

SSL was Netscape's 1990s protocol for encrypting web traffic. Every version of SSL is now broken and disabled. Its successor, TLS (Transport Layer Security), is what actually protects traffic today; TLS 1.0 and 1.1 are deprecated, TLS 1.2 is the minimum, and TLS 1.3 is the current version. People still say "SSL certificate" out of habit, but the protocol underneath is TLS.

HTTPS is simply HTTP running inside a TLS connection. Nothing about our API changes. Every request we have written in eight weeks would work the same over HTTPS; only the transport underneath is protected.

### What TLS does in a handshake

When a browser connects to `https://` a site, a short handshake happens before any HTTP is sent:

1. The browser says which TLS versions and ciphers it supports.
2. The server answers with its choice and its certificate: its public key, signed by a certificate authority (CA).
3. The browser checks the signature against CAs it already trusts and checks that the certificate's name matches the site.
4. Both sides use asymmetric key exchange to agree on a fresh symmetric session key.
5. Everything after that, including the HTTP request line, headers, cookies, and body, is encrypted with that key.

TLS gives three guarantees: confidentiality (eavesdroppers see noise), integrity (tampering is detected), and authentication (you are talking to the real site, not an impostor). Browsers are also starting to use post-quantum key exchange in the handshake, so traffic recorded today can't be decrypted by a future quantum computer.

### HSTS and secure cookies

Getting TLS on a server isn't enough by itself. A user who types `bookstore.example.com` still starts with plain HTTP, and an attacker on public Wi-Fi can intercept that first request. Three settings close the gaps:

- Redirect HTTP to HTTPS (`SECURE_SSL_REDIRECT` in Django).
- Send HSTS (HTTP Strict Transport Security), a header telling the browser to use HTTPS for this site from now on, without even trying HTTP (`SECURE_HSTS_SECONDS`).
- Mark cookies `Secure` so the browser never sends them over HTTP (`SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`). 

### Other protocols at other layers

TLS isn't the only way to secure traffic. The protocols differ in which layer they protect:

| Protocol | Layer | Protects | Typical use |
|---|---|---|---|
| TLS (HTTPS) | Transport, per connection | One application's traffic | Websites and APIs |
| IPSec | Network, per packet | All traffic between two networks or hosts | Site-to-site VPNs, remote access VPNs |
| SSH and SFTP | Application, per session | Shell access and file transfer | Deploying code, managing servers |
| FTP | None | Nothing; passwords travel in plain text | Avoid it |
| FTPS | FTP inside TLS | File transfer | Legacy systems that still need FTP |


## Testing Web Applications

So far we have tested by hand: run a client script, read the output, compare with the notes. That doesn't scale, and it doesn't happen again after every change. Automated tests do.

| Level | What it checks | Speed | Bookstore example |
|---|---|---|---|
| Unit | One function or class in isolation | Milliseconds | `Book.as_dict()` returns a float price |
| Integration | Several parts together, through real requests | Fast | POST to `/api/v2/books/` returns 400 for a negative price |
| End-to-end | The whole system through a real browser | Seconds | Fill in the shop form and see the new book in the list |

The usual advice is many unit tests, a fair number of integration tests, and a few end-to-end tests, because the higher levels are slower and break more easily.

## E-Commerce Law and Regulation

The moment the bookstore takes real orders, it collects personal data, takes payments, sends email, and makes promises to consumers. Each of those is regulated. The rules depend on where your customers are, not only where you are. This is an overview for developers, not legal advice; real projects need a lawyer.

| Rule | Applies to | What it asks of the website |
|---|---|---|
| GDPR (EU and UK) | Anyone processing personal data of people in the EU or UK | A lawful basis for each use of data, clear privacy notices, rights to access and delete, breach notification within 72 hours, privacy by design. Fines can reach 4% of global annual turnover |
| ePrivacy rules (EU) | Sites setting cookies for EU visitors | Consent before non-essential cookies (analytics, ads); essential ones like our session cookie are exempt |
| US state privacy laws | Businesses above size thresholds serving residents of those states | About 20 states now have comprehensive laws (California's CCPA/CPRA is the strictest); rights to know, delete, and opt out of the sale or sharing of data |
| FTC Act, Section 5 (US) | Nearly all US businesses | No unfair or deceptive practices: privacy policies must be true, security promises must be kept, and fake reviews are banned outright |
| ROSCA and auto-renewal laws (US) | Subscriptions | Clear terms before billing, express consent, and a simple way to cancel. The FTC's 2024 click-to-cancel rule was vacated by a federal court in 2025, and the FTC restarted the rulemaking in 2026; ROSCA and many state laws still apply |
| CAN-SPAM (US) | Commercial email | Honest subject lines and senders, a working unsubscribe, and opt-outs honored within 10 business days |
| COPPA (US) | Sites aimed at children under 13, or that knowingly collect their data | Verifiable parental consent before collecting children's data; updated rules took full effect in 2026 |
| PCI DSS v4.0.1 | Anyone who stores, processes, or transmits card data | An industry standard enforced by contract with card networks, not a law: secure networks, encryption, access control, monitoring, and, since 2025, controls on scripts running in payment pages |
| Accessibility (ADA in the US, European Accessibility Act in the EU) | Online stores serving the public | Usable by people with disabilities, measured in practice against WCAG; the EAA has applied to e-commerce since June 2025 |
| ESIGN and UETA (US), eIDAS (EU) | Online contracts | Electronic signatures and click-through agreements are legally valid |
| Breach notification laws | All 50 US states, plus GDPR | Tell affected people (and often regulators) promptly when personal data is exposed |

Two practical consequences for developers stand out. First, the easiest card data to protect is data you never touch. Using a payment provider's hosted checkout or payment fields keeps card numbers off your servers and shrinks your PCI DSS scope dramatically. Second, privacy by design is cheaper than privacy by retrofit: collect only the fields you need, decide how long to keep them, and build the delete function at the same time as the save function.

Notice how many of these laws require the technical controls from the first half of the lecture. GDPR expects "appropriate" security, PCI DSS requires TLS and logging, and the FTC has treated broken security promises as deception. The law and the code point the same way.

## Exercise 1: Attacking Our Own Bookstore

We'll attack the bookstore with four small experiments: SQL injection in the shell, stored XSS through the shop page, a CSRF check that switches on at login, and the information a debug server gives away. Each takes a few lines, and each one undoes itself.

Open a terminal in the `bookstore` project folder (the one with `manage.py`) and activate your virtual environment. Use Chrome or Edge for the browser steps.

### Step 1: Start from a known state

Stop the server if it is running (Ctrl+C), reset the data, and create an admin account. Pick any username and password when prompted.

```bash
python manage.py flush --no-input
python manage.py seed_books
python manage.py createsuperuser
```

The `seed_books` line prints:

```text
Total books in database: 3
```

`flush` empties every table and resets the ID counter, so the next book will be ID 4. It also removes any admin account from earlier, which is why `createsuperuser` comes after it.

### Step 2: SQL injection in the shell

Open the Django shell with `python manage.py shell`. Paste this block on its own and press Enter on the blank line after the function:

```python
from catalog.models import Book

def find_in_stock(title):
    query = f"SELECT * FROM catalog_book WHERE in_stock = 1 AND title = '{title}'"
    return [book.title for book in Book.objects.raw(query)]

```

Then run these three lines:

```python
print(find_in_stock("Fluent Python"))
print(find_in_stock("Clean Code"))
print(find_in_stock("nothing' OR '1'='1"))
```

```text
['Fluent Python']
[]
['Django 5 by Example', 'Fluent Python', 'Clean Code']
```

The function promises to return only books that are in stock. Clean Code is out of stock, so the second call correctly returns nothing. The third "title" changes the query itself. After pasting, the database sees `WHERE in_stock = 1 AND title = 'nothing' OR '1'='1'`. `AND` binds tighter than `OR`, so the condition becomes "(in stock and titled nothing) or true", which matches every row, and the hidden book comes back. Now picture the same query in a real store with `customer_id = 42` in place of `in_stock = 1`. The attacker would see every customer's orders, not just their own. Injection lets user input rewrite the rules of a query, and those rules are often all that separates one user's data from another's.

Now the fix. Paste this block on its own and press Enter on the blank line after it:

```python
def find_in_stock(title):
    query = "SELECT * FROM catalog_book WHERE in_stock = 1 AND title = %s"
    return [book.title for book in Book.objects.raw(query, [title])]

```

Then run:

```python
print(find_in_stock("Fluent Python"))
print(find_in_stock("nothing' OR '1'='1"))
```

```text
['Fluent Python']
[]
```

With `%s`, the database receives the query and the value separately, so the quote is just a character in a title nobody has. The rule `in_stock = 1` can no longer be removed. The ORM does this for you: `Book.objects.filter(in_stock=True, title=title)` is always safe. Type `exit()` to leave the shell.

### Step 3: Stored XSS through the shop page

Start the server with `python manage.py runserver`. In `catalog/static/catalog/shop.js`, find this line in `loadBooks`:

```js
      item.textContent = `${book.title} by ${book.author}, $${book.price.toFixed(2)}`;
```

and change `textContent` to `innerHTML`, as a careless developer might to "allow formatting":

```js
      item.innerHTML = `${book.title} by ${book.author}, $${book.price.toFixed(2)}`;
```

Visit `http://127.0.0.1:8000/shop/`, press Ctrl+F5, and add a book with this title, `Mallory` as the author, and `1` as the price:

```text
<img src=x onerror="alert('XSS')">
```

An alert box saying `XSS` pops up as soon as the list redraws, and the new list item reads only ` by Mallory, $1.00` next to a broken image icon. The title became a real `<img>` tag; its image failed to load, and the browser ran the `onerror` script. Now open the same page in a second tab: the alert fires again. The script is stored in the database, so it runs for every visitor, and in a real attack it would send their cookies or session data somewhere instead of showing an alert.

Change `innerHTML` back to `textContent` and press Ctrl+F5. The item now shows the attack as harmless text:

```text
<img src=x onerror="alert('XSS')"> by Mallory, $1.00
```

The malicious title is still in the database. Fixing the output, not hunting down bad input, is what made it safe, and it protects against every payload, including ones nobody has thought of yet.

### Step 4: CSRF and missing access control

Leave the shop page open, press F12, and choose the Console tab. Paste this and press Enter:

```js
let r = await fetch("/api/v2/books/", {
  method: "POST",
  headers: {"Content-Type": "application/json"},
  body: JSON.stringify({title: "No Token", author: "Anon", price: 1}),
});
console.log(r.status, await r.text());
```

Expected output:

```text
201 {"id":5,"title":"No Token","author":"Anon","price":1.0,"in_stock":true}
```

No CSRF token, no login, and the book was created anyway. Two lessons are hiding here. DRF skips the CSRF check because no one is logged in, so there is no session for an attacker to borrow. And the API has no access control at all: anyone on the internet can add books. That is A01, Broken Access Control, the top risk on the list.

Now log in. Visit `http://127.0.0.1:8000/admin/`, sign in with the account from Step 1, then go back to `/shop/`, press Ctrl+F5, and paste the same block in the Console again:

```text
403 {"detail":"CSRF Failed: CSRF token missing."}
```

The browser now sends a `sessionid` cookie for a logged-in user, so DRF turns on the CSRF check, and a request without the token looks exactly like a forged one. The page's own form still works, because it sends the `X-CSRFToken` header from Week 7. Add a book called `With Token` by `Admin` for `2`:

```text
Added "With Token" with ID 6.
```

Log out of the admin (Log out at the top right) before moving on.

### Step 5: What a misconfigured server gives away

Visit `http://127.0.0.1:8000/api/nope/`. Instead of a plain "Not Found", Django shows a yellow debug page listing every URL pattern in the project, from `admin/` to `api/ stats/`. With `DEBUG = True`, error pages also show stack traces, settings, and local variables. That is a map of the application handed to anyone who mistypes a URL.

Django can audit its own settings. Stop the server and run:

```bash
python manage.py check --deploy
```

You'll get seven security warnings. The IDs and what each one means:

| Warning | Problem |
|---|---|
| `security.W004` | No HSTS (`SECURE_HSTS_SECONDS` not set) |
| `security.W008` | HTTP isn't redirected to HTTPS (`SECURE_SSL_REDIRECT`) |
| `security.W009` | `SECRET_KEY` is the generated `django-insecure-` one |
| `security.W012` | Session cookie can travel over HTTP (`SESSION_COOKIE_SECURE`) |
| `security.W016` | CSRF cookie can travel over HTTP (`CSRF_COOKIE_SECURE`) |
| `security.W018` | `DEBUG` is `True` |
| `security.W020` | `ALLOWED_HOSTS` is empty |

On Django 6.1 or newer, you'll also see an error `mail.E001` about the development email backend. We don't send email, so ignore it.

## Exercise 2: Testing and Hardening

We'll write automated tests for the defenses we just saw, then use a failing test to finally fix the duplicate POST from Weeks 6 and 7.

### Step 1: Write the tests

`startapp` created an almost empty `catalog/tests.py` back in Week 3. Replace its contents with:

```python
# catalog/tests.py
from django.contrib.auth.models import User
from django.test import Client, TestCase

from .models import Book

NEW_BOOK = {"title": "Refactoring", "author": "Martin Fowler", "price": "47.99"}


class BookAPITests(TestCase):
    def setUp(self):
        Book.objects.create(title="Clean Code", author="Robert C. Martin", price="39.99")

    def post_book(self, client, data):
        return client.post("/api/v2/books/", data, content_type="application/json")

    def test_negative_price_is_rejected(self):
        response = self.post_book(self.client, {**NEW_BOOK, "price": "-5"})
        self.assertEqual(response.status_code, 400)
        self.assertEqual(Book.objects.count(), 1)

    def test_logged_in_post_without_csrf_token_is_refused(self):
        User.objects.create_user("alice", password="test-pass-123")
        client = Client(enforce_csrf_checks=True)
        client.login(username="alice", password="test-pass-123")
        response = self.post_book(client, NEW_BOOK)
        self.assertEqual(response.status_code, 403)

    def test_duplicate_book_is_rejected(self):
        first = self.post_book(self.client, NEW_BOOK)
        second = self.post_book(self.client, NEW_BOOK)
        self.assertEqual(first.status_code, 201)
        self.assertEqual(second.status_code, 400)
        self.assertEqual(Book.objects.filter(title="Refactoring").count(), 1)
```

`TestCase` gives each test a clean slate: `setUp` runs before every test and creates one book in a separate test database, and everything is rolled back after the test finishes. Every method whose name starts with `test_` is one test, and each `assert...` line is a check that fails the test if it isn't true. `self.client` is Django's built-in test client, which calls our views directly, without a server.

The three tests turn three promises into code. The first checks that the server rejects bad input even though no browser is involved. The second repeats Exercise 1 Step 4: a logged-in user without a token gets 403. It needs `Client(enforce_csrf_checks=True)`, because the default test client skips CSRF checks. The third says what should happen to a duplicate: the second identical POST is refused, and only one Refactoring exists.

### Step 2: Run them and watch one fail

```bash
python manage.py test catalog
```

Expected output, shortened (`F` marks the failed test, a dot each passed one; the order may differ):

```text
Found 3 test(s).
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
F..
======================================================================
FAIL: test_duplicate_book_is_rejected (catalog.tests.BookAPITests.test_duplicate_book_is_rejected)
----------------------------------------------------------------------
Traceback (most recent call last):
  ...
    self.assertEqual(second.status_code, 400)
AssertionError: 201 != 400

----------------------------------------------------------------------
Ran 3 tests in 0.937s

FAILED (failures=1)
Destroying test database for alias 'default'...
```

Two tests pass: the validation and the CSRF check are already working. The third fails, and that's on purpose: it describes the behavior we want, not the behavior we have. The server still happily saves the same book twice, exactly as Week 6's second client run and Week 7's double click showed.

### Step 3: Fix it in the database

Open `catalog/models.py` and add a `constraints` list under `ordering` in the `Meta` class:

```python
    class Meta:
        ordering = ["id"]
        constraints = [
            models.UniqueConstraint(fields=["title", "author"], name="unique_title_author"),
        ]
```

Then create and apply the migration, and run the tests again:

```bash
python manage.py makemigrations catalog
python manage.py migrate
python manage.py test catalog
```

```text
Migrations for 'catalog':
  catalog/migrations/0002_book_unique_title_author.py
    + Create constraint unique_title_author on model book
...
  Applying catalog.0002_book_unique_title_author... OK
...
Ran 3 tests in 0.852s

OK
```

A unique constraint lives in the database itself, so no client, script, retry, or second tab can get around it. The same title by a different author is still allowed. If `migrate` fails with `UNIQUE constraint failed`, your database still has duplicates from an earlier week; run Exercise 1 Step 1 again and retry.

We changed only the model, yet v2 now answers a duplicate with a clean 400. `ModelSerializer` reads the constraint and adds a matching validator, so the error arrives in the same format as every other validation error. Start the server, open `/shop/`, and add Refactoring by Martin Fowler for `47.99` twice. The second time the form shows:

```text
non_field_errors: The fields title, author must make a unique set.
```

Week 7 ended with "the client makes mistakes unlikely; only the server can make them impossible." The disabled button was the first half. This constraint is the second, and the test makes sure it stays that way.

## Bonus Activity (Ungraded, At Home)

1. Week 5's `cart_view` accepts zero and negative quantities. Write a test that POSTs `{"book_id": 1, "quantity": -3}` and expects 400, see it fail, and then fix the view.

## References

- Django 5 by Example: Build Powerful and Reliable Python Web Applications from Scratch
    - Chapter 17: Going Live
- Django Software Foundation. (n.d.). Security in Django. Django documentation. https://docs.djangoproject.com/en/stable/topics/security/
- Django Software Foundation. (n.d.). Testing tools. Django documentation. https://docs.djangoproject.com/en/stable/topics/testing/tools/
