# Week 2: Web Frameworks, MVC and Multi-Tier (N-Tier) Architecture

## 1. From Week 1 to a Real Problem

Last week we got a view and a form working, but everything lived in one
file inside the mysite config package: request handling, the response
text, all of it tangled together. That was fine for a two-view demo. It
falls apart the moment a project grows past a handful of pages, because
there's no separation between what the app does, what it stores, and
what the user sees.

MVC is the answer to that problem. It's not specific to Django or even to
web development. It's a way of organizing any application with a user
interface.

## 2. What Is MVC?

Model, View, Controller splits an application into three roles:

- Model: the data and the business rules around it. Owns the "what."
- View: what gets rendered to the user. Owns the "how it looks."
- Controller: receives the request, talks to the Model, picks a View,
  returns a response. Owns the "what happens."

The point isn't the three names. It's that each piece can change on its
own. A template can be redesigned without touching business logic. A
database schema can change without rewriting the UI. That independence is
what makes a codebase maintainable past week one of a project.

## 3. N-Tier Architecture

N-tier is the same idea applied at a larger scale, often across physical
or network boundaries rather than just files in a project:

- Presentation tier: what the client sees and interacts with
- Application (logic) tier: where requests get processed and decisions
  get made
- Data tier: where information is stored and retrieved

MVC is one common way to implement the presentation and application
tiers of an N-tier system. The View corresponds to the presentation tier.
The Controller lives in the application tier, coordinating between the
View and the Model. The Model corresponds to the data tier, which we'll
build out properly in Week 6 with the ORM.

## 4. Django's Flavor: MVT

Django calls its pattern MVT: Model, View, Template. The names don't line
up cleanly with classic MVC, and that trips people up, so here's the
mapping:

| Classic MVC | Django MVT | Role |
|---|---|---|
| Controller | View (views.py) | Receives the request, decides what happens |
| View | Template | Renders the output |
| Model | Model | Data and business rules (Week 6) |

Django's "View" is doing the Controller's job. Django's "Template" is
doing the classic View's job. Once that terminology clicks, everything
else in the framework reads consistently.

This naming shuffle is the point worth remembering past this course.
Every mainstream WAF implements MVC, just with its own vocabulary:

| Framework | Controller equivalent | View equivalent |
|---|---|---|
| Django | views.py functions | Templates |
| Spring Boot (Java) | @Controller classes | Thymeleaf/JSP templates |
| Express (Node) | Route handler functions | EJS/Handlebars templates |
| ASP.NET Core | Controller classes | Razor views |

Same three roles, same reasons for separating them, different names and
syntax. That's what CLO 2 is actually asking you to recognize: the
pattern, not the Django API for it.

## 5. Django Apps vs the Project

A Django project is the whole site: settings, top-level URL routing,
deployment configuration. An app is a self-contained bundle of related
functionality, meant to be focused and, ideally, reusable. A real project
is usually one project with several apps: a blog app, an accounts app, a
payments app. This is Django's version of a general practice: every WAF
gives you some way to break a growing codebase into cohesive, independent
modules, whether that's a Spring module, an Express router, or an
ASP.NET area. The layout and naming change; the reason you do it doesn't.

Last week we put our view directly in the mysite config package as a
deliberate shortcut. That package is meant for project-wide
configuration, not application code. Today we fix that.

### Creating an app

```bash
python manage.py startapp core
```

This generates a core/ folder with its own views.py, models.py,
migrations/, and an apps.py. Compare that to mysite/, which only has
settings.py, urls.py, and the WSGI/ASGI entry points. Different jobs,
different structure.

### Registering the app

In mysite/settings.py, add the app to INSTALLED_APPS:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "core",
]
```

Django won't pick up an app's templates, models, or admin registrations
until it's listed here.

### Giving the app its own routes

Create core/urls.py:

```python
from django.urls import path
from . import views

urlpatterns = [
    path("hello/", views.hello),
    path("contact/", views.contact),
]
```

Then in mysite/urls.py, hand off to it with include():

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("core.urls")),
]
```

The project's urls.py now only knows that core handles some set of
routes. It doesn't need to know which ones. That's the separation doing
its job already.

## 6. Where Templates Live Now

App-level templates go in a nested folder: core/templates/core/. The
extra core/ inside templates/ looks redundant with one app, but it
namespaces the template so that render(request, "core/form.html") can't
collide with another app's form.html once you have more than one. This
is a Django convention worth adopting from day one.

## 7. The Django Template Language

Templates use two kinds of markup:

- {{ variable }} outputs a value from the context
- {% tag %} handles logic: loops, conditionals, template inheritance

The language is deliberately limited. You can loop and branch, but you
can't do arbitrary computation. That's intentional: logic belongs in the
view, not the template. Keeping that boundary is most of what makes MVC
actually work in practice instead of just being a folder structure.

```html
{% if user_count > 0 %}
    <p>{{ user_count }} users online</p>
{% else %}
    <p>No one's here.</p>
{% endif %}
```

## 8. Template Inheritance

Repeating the same HTML skeleton (head, nav, footer) in every template is
exactly the kind of duplication MVC is meant to prevent. Django handles
this with a base template and a block tag.

base.html:

```html
<!DOCTYPE html>
<html>
<head><title>{% block title %}My Site{% endblock %}</title></head>
<body>
    <header>Core App</header>
    {% block content %}{% endblock %}
</body>
</html>
```

A child template extends it and only fills in the block:

```html
{% extends "core/base.html" %}
{% block title %}Contact{% endblock %}
{% block content %}
    <h1>Contact us</h1>
{% endblock %}
```

## 9. Data Grids: Rendering Lists

A data grid is just a list of records rendered as rows, one of the most
common things a web app does. We're not touching a real database until
Week 6, so today the "records" are a plain Python list of dictionaries.
The pattern is the same either way: the view decides what data goes into
the context, the template loops over it and decides how it looks.

```python
def products(request):
    items = [
        {"name": "Widget", "price": 9.99},
        {"name": "Gadget", "price": 19.99},
        {"name": "Gizmo", "price": 14.50},
    ]
    return render(request, "core/products.html", {"items": items})
```

```html
{% extends "core/base.html" %}
{% block content %}
<table>
    <tr><th>Name</th><th>Price</th></tr>
    {% for item in items %}
    <tr>
        <td>{{ item.name }}</td>
        <td>${{ item.price }}</td>
    </tr>
    {% endfor %}
</table>
{% endblock %}
```

Same dot syntax works for dictionary keys and object attributes, which is
why this pattern will still apply once items comes from the ORM instead
of a hardcoded list.

Strip away the Django syntax and what's happening is: the controller
tier assembles data, hands it to the presentation tier, and the
presentation tier is the only place that knows about rows and columns.
A Spring controller returning a model to a Thymeleaf template, or an
Express route rendering an EJS view, follows the identical division of
labor. The syntax you're typing today is Django's, but the design
decision you're practicing is the one CLO 2 covers.

---

## Exercise: Refactor Into an App, Add a Data Grid

Goal: move last week's code out of the mysite config package and into a
proper app, then add a new view that renders a data grid.

### Step 1: Create the app

```bash
python manage.py startapp core
```

### Step 2: Move the views

Move hello and contact from mysite/views.py into core/views.py. Update
the render() call inside contact to use the namespaced path:

```python
from django.http import HttpResponse
from django.shortcuts import render

def hello(request):
    return HttpResponse("Hello from the server. Method used: " + request.method)

def contact(request):
    if request.method == "POST":
        message = request.POST.get("message")
        return HttpResponse(f"You sent: {message}")
    return render(request, "core/form.html")
```

### Step 3: Move the template

Move templates/form.html into core/templates/core/form.html.

### Step 4: Register the app and wire routing

In mysite/settings.py, add "core" to INSTALLED_APPS:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "core",
]
```

Create core/urls.py:

```python
from django.urls import path
from . import views

urlpatterns = [
    path("hello/", views.hello),
    path("contact/", views.contact),
]
```

Replace the contents of mysite/urls.py so the project hands routing off
to the app:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("core.urls")),
]
```

### Step 5: Confirm the refactor didn't break anything

Run the server and revisit /hello/ and /contact/. Same behavior as last
week, different file layout. If something 404s, it's almost always a
missing INSTALLED_APPS entry or a template Django can't find because of
the namespacing folder.

```bash
python manage.py runserver
```

### Step 6: Add the data grid

Add a products view to core/views.py:

```python
def products(request):
    items = [
        {"name": "Widget", "price": 9.99},
        {"name": "Gadget", "price": 19.99},
        {"name": "Gizmo", "price": 14.50},
    ]
    return render(request, "core/products.html", {"items": items})
```

Create core/templates/core/base.html, the shared layout every page in
this app will extend:

```html
<!DOCTYPE html>
<html>
<head><title>{% block title %}My Site{% endblock %}</title></head>
<body>
    <header>Core App</header>
    {% block content %}{% endblock %}
</body>
</html>
```

Create core/templates/core/products.html:

```html
{% extends "core/base.html" %}
{% block content %}
<table>
    <tr><th>Name</th><th>Price</th></tr>
    {% for item in items %}
    <tr>
        <td>{{ item.name }}</td>
        <td>${{ item.price }}</td>
    </tr>
    {% endfor %}
</table>
{% endblock %}
```

Route it in core/urls.py:

```python
urlpatterns = [
    path("hello/", views.hello),
    path("contact/", views.contact),
    path("products/", views.products),
]
```

Visit /products/. This is the first view where the template is rendering
something the view computed, rather than just static markup. That's the
MVC split working end to end.

---

## Bonus Activity (Ungraded)

Extend the products view:

1. Add a fourth item to the items list with a price that would sort
   differently.
2. Sort the list by price before passing it to the template. Notice this
   sorting decision belongs in the view, not the template. That's the
   line to internalize before Week 6, when items stops being a Python
   list and starts being a database query.

This doesn't need to be submitted.

## References
- Django 5 by Example: Build Powerful and Reliable Python Web
  Applications from Scratch
  - Chapter 1: Building a Blog Application
  - Chapter 2: Enhancing Your Blog with Advanced Features
  - Chapter 3: Extending Your Blog Application
- [Django Template Language docs](https://docs.djangoproject.com/en/5.0/ref/templates/language/)
