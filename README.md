# WOLFx Admin Configuration - Unfold Edition

Drop-in `admin.py` that registers every model in your app automatically and takes
its entire configuration from an `admin_meta` dictionary on the model itself.

This is the [django-unfold](https://github.com/unfoldadmin/django-unfold) branch.
It adds the Unfold UI, import/export, a rich text editor, and a JSON schema
editor on top of the auto-registration in `main`.

> Looking for plain Django admin with no extra dependencies? Use the `main` branch.

## Why

A typical Django project describes each model twice: once in `models.py` and
again in a `ModelAdmin`. The two drift, and the drift is where bugs live.

Here the model is the only description. Add a field and it shows up in the admin.
Add a model and it registers itself. The configuration sits next to the fields it
configures.

## Install

1. Install the dependencies:

```bash
pip install django-unfold django-import-export
```

2. Add Unfold to `INSTALLED_APPS`, **before** `django.contrib.admin`:

```python
INSTALLED_APPS = [
    'unfold',
    'unfold.contrib.filters',
    'unfold.contrib.forms',
    'unfold.contrib.import_export',

    'django.contrib.admin',
    ...
    'import_export',
]
```

3. Copy `admin.py` into your app.

4. The app also needs a `resources.py` and a `widgets.py`, since `admin.py`
   imports from both. Create empty ones if you have nothing to put in them yet:

```bash
touch your_app/resources.py your_app/widgets.py
```

`widgets.py` must define `JsonEditorWidget` if you plan to use the `json_fields`
key. A working implementation is in the `main` branch's `admin.py`.

That is the whole setup. Every model in the app is now in the admin.

## Configuration constants

At the top of `admin.py`:

- `exempt`: lowercase model names that should not be registered. Defaults to
  `['user']` because the file registers the user model explicitly with Unfold's forms.
- `global_app_name`: derived from the directory containing `admin.py`, so the
  same file works in any app without editing.
- `resource_class_mapping`: maps a model to a custom django-import-export
  `Resource`. Models without an entry get a generated one, so import and export
  work everywhere by default.

Registration is skipped for models in `exempt`, django-simple-history shadow
tables (any name containing `histor`), and the through models Django
auto-creates for `ManyToManyField`s.

## The `admin_meta` dictionary

Every key that matches a `ModelAdmin` attribute is applied directly, because
`GenericAdmin.__init__` copies them onto the admin instance:

```python
class Blog(CommonModel):
    title       = models.CharField(max_length=200)
    text        = models.TextField(null=True, blank=True)
    head        = models.TextField(null=True, blank=True)
    category    = models.ForeignKey(BlogCategory, null=True, on_delete=models.SET_NULL)

    admin_meta = {
        'list_display'          : ('__str__', 'category', 'created_at'),
        'list_editable'         : ('category',),
        'list_filter'           : ('category',),
        'search_fields'         : ('title', 'category__category'),
        'autocomplete_fields'   : ('category',),
        'list_per_page'         : 50,
        'ordering'              : ('order_by',),
        'readonly_fields'       : ('slug',),
        'rtf_fields'            : ['text'],
    }
```

So `list_display`, `list_filter`, `search_fields`, `ordering`, `list_editable`,
`list_per_page`, `readonly_fields`, `autocomplete_fields`, `fieldsets` and the
rest all behave exactly as they do on a normal `ModelAdmin`.

### Special keys

These are handled by this file rather than passed through to Django.

| Key | Effect |
| --- | --- |
| `rtf_fields` | TextFields that get the rich text editor. Opt-in. |
| `rtf_exclude` | Legacy opt-out list. Still honoured, see below. |
| `single_entry` | Hides the Add button once one row exists. For singletons. |
| `json_fields` | Renders a JSONField with a schema-driven JSON editor. |
| `inline` | Registers related models as inlines. |
| `actions` | Binds model methods as admin actions. |
| `foreignkey_filters` | Inlines only. Filters a FK queryset against the parent. |

## Rich text is opt-in

A `TextField` renders as a plain textarea unless the model lists it in
`rtf_fields`:

```python
admin_meta = {
    'rtf_fields': ['description'],   # only this field gets the editor
}
```

This default is deliberate. The editor rewrites its contents on save, so pointing
it at a field holding raw HTML, a `<head>` block, a URL trace, or any other
machine-read value will silently corrupt that value the first time someone opens
and saves the record. Opt in only for authored prose.

### Migrating from `rtf_exclude`

Earlier versions were opt-*out*: every `TextField` got the editor except the ones
listed in `rtf_exclude`. That key still works, so existing `admin_meta` dicts
behave exactly as before and need no changes.

Resolution order:

1. `rtf_fields` declared, so only those fields get the editor.
2. `rtf_exclude` declared, so every TextField except those. Legacy.
3. Neither declared, so plain textareas everywhere.

If a model declares both, `rtf_fields` wins.

**When you migrate a model, list every field that should keep its editor.** Under
`rtf_exclude` an unlisted TextField got the editor implicitly. Under `rtf_fields`
an unlisted field gets nothing, so a straight deletion of `rtf_exclude` silently
removes editors you wanted to keep:

```python
# before
'rtf_exclude': ['head', 'tags']          # text and featured_text were rich

# after
'rtf_fields' : ['text', 'featured_text'] # name them explicitly
```

## Inlines

Declare them on the parent, mapping the related model name to the foreign key
field on that related model:

```python
admin_meta = {
    'inline': [{'BlogImage': 'blog'}],
}
```

The related model must live in the same app. Its own `admin_meta` is applied to
the inline, so `list_display`, `rtf_fields`, and the rest carry over.

Inlines also support `foreignkey_filters`, which narrows a FK dropdown based on
the parent object being edited:

```python
admin_meta = {
    'foreignkey_filters': {
        'room': lambda request, parent: Room.objects.filter(villa=parent),
    },
}
```

## Actions

Define an ordinary instance method that takes `request`, then name it in
`admin_meta`:

```python
class Order(CommonModel):
    admin_meta = {
        'actions': ['mark_as_shipped'],
    }

    def mark_as_shipped(self, request):
        self.status = 'shipped'
        self.save(update_fields=['status'])
    mark_as_shipped.short_description = "Mark selected orders as shipped"
```

The admin calls it **once per selected object**, as `obj.mark_as_shipped(request)`.
It is not a bulk queryset action, so write it for a single instance. That also
keeps the method callable from a shell, a signal, or a management command.

## JSON fields

```python
admin_meta = {
    'json_fields': {
        'data': {
            'schema': {
                'type': 'object',
                'properties': {
                    'key'  : {'type': 'string'},
                    'value': {'type': 'number'},
                },
            }
        }
    },
}
```

Requires `JsonEditorWidget` in your app's `widgets.py`. It wraps
[json-editor](https://github.com/json-editor/json-editor). Without a schema the
field stays a normal textarea.

## Fieldsets

With no `fieldsets` key, the admin builds two tabs: the model's own editable
fields, then a "Meta Data" tab holding the `CommonModel` bookkeeping fields
(`created_at`, `updated_at`, `extra_params`, and so on).

Declaring `fieldsets` replaces this entirely, including the Meta Data tab, so
list every field you still want visible:

```python
admin_meta = {
    'fieldsets': [
        ('General', {'classes': ['tab'], 'fields': ['logo', 'favicon']}),
        ('Social',  {'classes': ['tab'], 'fields': ['facebook', 'instagram']}),
    ],
}
```

## CommonModel

`CommonModel` is optional. If your app defines one, this file detects it and
groups its fields into the Meta Data tab and pushes them to the end of inline
forms. Without it everything still works, the fields just are not grouped.

A typical one:

```python
class CommonModel(models.Model):
    id           = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    extra_params = models.JSONField(blank=True, null=True)
    created_at   = models.DateTimeField(auto_now_add=True, blank=True, null=True)
    updated_at   = models.DateTimeField(auto_now=True, blank=True, null=True)
    created_by   = models.CharField(max_length=300, blank=True, null=True)
    updated_by   = models.CharField(max_length=300, blank=True, null=True)

    admin_meta   = {}

    class Meta:
        abstract = True
```

## Changelog

- Rich text is now opt-in via `rtf_fields`. `rtf_exclude` still works.
- `readonly_fields` in `admin_meta` is now applied. It was previously overwritten
  by `get_readonly_fields` and had no effect.
- The user and group models are registered with Unfold's styled forms, including
  the password change form.
- Auto-created M2M through models are no longer registered as their own entries.
- Fixed an `UnboundLocalError` in inline `get_formset` when no `CommonModel` exists.
- A single-string field list such as `('head')` is no longer matched character by
  character.

## What is WOLFx?

[WOLFx Digital Agency](https://wolfx.io) is an IT development company based in
Mumbai, India, building custom software, web and mobile applications, and
providing IT consulting and digital transformation services. We contribute to
the open-source community alongside our commercial work.

## Contribution

Issues, feature requests, and pull requests are welcome.

## License

MIT. See the LICENSE file for details.
