# Welcome to the WOLFx Admin Configuration repository! 🚀

WOLFx Admin configuration is created to improve the developer experience by simplifying the process of registering models in the admin site. This powerful tool is designed to automate admin registrations but it does not compromise the features that Django's default admin site provides. You can control the admin interface directly from your `models.py` file using a simple `admin_meta` dictionary. This approach eliminates the need for repetitive tasks associated with traditional model registration and allows for direct control over the admin interface, keeping your configurations closely tied to your model definitions.

This is the **`main`** branch: plain Django admin, no third-party dependencies.

> Want the Unfold UI, import/export, and a rich text editor? Use the
> [`django-unfold-admin`](../../tree/django-unfold-admin) branch.

## Key Features

1. **Automated Model Registration**: Automatically register your models in the Django admin site without the hassle of manually creating admin classes.
2. **Direct Control from Models**: Manage your admin configurations right next to your model definitions using the `admin_meta` dictionary.
3. **Maintains Django Admin Features**: Retain all the robust features offered by Django's default admin site while enjoying the benefits of simplified management.
4. **No dependencies**: This branch uses nothing but Django itself.

## Usage

1. Place the code in the `admin.py` file of your Django application.
2. Adjust the configuration constants if you need to.
3. Define `admin_meta` in your models to customize the admin interface.

That is it. Every model in the app is now registered.

## NOTE: Configuration Constants 📌

- `admin.site.site_header`: Sets the header for the admin site.
- `exempt`: A list of model names that should not be registered with the admin. Note that the `model_name` should be used and not the model itself. To see the actual `model_name` of your model, uncomment the `print(model_name + ' ' + str(model))` near the end of the file.
- `global_app_name`: The app to pull models from. It is **derived automatically** from the directory containing `admin.py`, so the same file works in any app without editing. Override it with a literal string only if your `admin.py` does not sit at the app package root.

Registration is skipped for three groups: names listed in `exempt`, django-simple-history shadow tables (any name containing `histor`), and the through models Django auto-creates for `ManyToManyField`s.

## Example Model Configuration 🛠️

Here's an example of how you can define your models to leverage the dynamic admin interface:

```python
class BlogCategory(CommonModel):
    category    =   models.CharField    (max_length=100, unique=True)
    slug        =   models.SlugField    (max_length=100, unique=True)
    image       =   models.FileField    (blank=True,null=True,upload_to='blog_category/')

    def __str__(self):
        return str(self.category)

class Blog(CommonModel):

    head_default='''<meta name="title" content=" ">
    <meta name="description" content=" ">
    <meta name="keywords" content=" ">
    <meta name="robots" content="index, follow">'''

    title               =   models.CharField        (max_length=200)
    sub_title           =   models.CharField        (max_length=200, blank=True ,null=True)
    thumbnail           =   models.ImageField       (upload_to="blog/")
    category            =   models.ForeignKey       (BlogCategory, null=True, on_delete=models.SET_NULL)
    featured_text       =   models.TextField        (null=True, blank=True)
    text                =   models.TextField        (null=True, blank=True)
    slug                =   models.SlugField        (unique=True)
    readtime            =   models.CharField        (max_length=200,null=True, blank=True)
    tags                =   models.TextField        (null=True, blank=True, default='all')
    head                =   models.TextField        (null=True, blank=True, default=head_default)

    order_by            =   models.IntegerField     (default=0)

    created_at          =   models.DateTimeField    (auto_now_add=True, blank=True, null=True)
    updated_at          =   models.DateTimeField    (auto_now=True, blank=True, null=True)
    created_by          =   models.CharField        (max_length=300)

    admin_meta =    {
        'list_display'      :   ("__str__","category","created_at","updated_at"),
        'list_editable'     :   ("category",),
        'list_per_page'     :   50,
        'list_filter'       :   ("category",),
        'readonly_fields'   :   ("slug",),
        'inline'            :   [
            {'BlogImage': 'blog'}
        ]
    }

    def __str__(self):
        return str(self.title)

    class Meta:
        verbose_name_plural = "Blog"
        ordering = ['order_by'] #Sort in desc order

class BlogImage(CommonModel):
    blog                =   models.ForeignKey       (Blog, on_delete=models.CASCADE)
    image               =   models.ImageField       (upload_to="blog_images/")
    order_by            =   models.IntegerField     (default=0)

    def __str__(self):
        return str(self.blog)

    class Meta:
        verbose_name_plural = "Blog Image"
        ordering = ['order_by'] #Sort in desc order
```

### Using the `admin_meta` Dictionary

You can control the admin interface directly from your `models.py` using the `admin_meta` dictionary.

Any key matching a `ModelAdmin` attribute is applied directly, so it behaves exactly as it would on a hand-written admin class:

- **`list_display`**: Fields to display in the list view.
- **`list_editable`**: Fields editable directly in the list view.
- **`list_filter`**: Fields to filter the list view.
- **`search_fields`**: Fields to search across.
- **`ordering`**, **`list_per_page`**, **`readonly_fields`**, **`autocomplete_fields`**, **`fieldsets`**: all supported.

These keys are handled by this file rather than passed through to Django:

| Key | Effect |
| --- | --- |
| `single_entry` | Hides the Add button once one row exists. For settings singletons. |
| `inline` | Registers related models as inlines. |
| `actions` | Binds model methods as admin actions. |
| `json_fields` | Renders a JSONField with a schema-driven JSON editor. |

An inline's own model `admin_meta` is applied to the inline too, so a related model configures its own inline presentation.

### Actions

Define an ordinary instance method that takes `request`, then name it in `admin_meta`:

```python
class Order(CommonModel):
    admin_meta = {
        'actions': ['mark_as_shipped'],
    }

    def mark_as_shipped(self, request):
        self.status = 'shipped'
        self.save(update_fields=['status'])
```

The admin calls it **once per selected object**, as `obj.mark_as_shipped(request)`. It is not a bulk queryset action, so write it for a single instance.

### Fieldsets

With no `fieldsets` key, the admin builds two groups: the model's own editable fields, then a "Meta Data" group holding the `CommonModel` bookkeeping fields. Declaring `fieldsets` replaces this entirely, including the Meta Data group, so list every field you still want visible.

### Example with JSONField Schema

If you have a JSONField in your model, you can customize its form field using a schema. This makes use of another [open-source project](https://github.com/json-editor/json-editor):

```python
class MyModel(models.Model):
    name = models.CharField(max_length=255)
    data = models.JSONField()

    admin_meta = {
        'json_fields': {
            'data': {
                'schema': {
                    'type': 'object',
                    'properties': {
                        'key': {'type': 'string'},
                        'value': {'type': 'number'}
                    }
                }
            }
        },
        'fieldsets': (
            (None, {
                'fields': ('name', 'data')
            }),
        ),
        'actions': ['custom_action'],
        'inline': [
            {'RelatedModel': 'mymodel'}
        ]
    }

    def custom_action(self, request):
        # Custom action logic
        pass
```

Without a schema the field stays a normal textarea.

### JSON Editor Widget

The `JsonEditorWidget` class is included in `admin.py` and renders JSON fields with a user-friendly editor. It loads [json-editor](https://github.com/json-editor/json-editor) from a CDN, so it needs no pip install.

## CommonModel

`CommonModel` is optional. When your app defines one, this file detects it, groups its fields into the "Meta Data" fieldset, and pushes them to the end of inline forms. Without it everything still works, the fields just are not grouped.

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

- `global_app_name` is derived from the app directory, so `admin.py` no longer needs editing per app. Set it explicitly to override.
- `readonly_fields` in `admin_meta` is now applied. It was previously rebuilt by `get_readonly_fields` and silently discarded.
- Added `single_entry`, matching the unfold branch.
- Inline field reordering actually works. It compared field name strings against `Field` objects, so it never matched and the reorder was a no-op.
- An inline now receives its own model's `admin_meta`.
- Auto-created M2M through models are no longer registered as their own admin entries.
- Bare `except:` clauses no longer swallow `KeyboardInterrupt` and `SystemExit`.

## What is WOLFx?

[WOLFx Digital Agency](https://wolfx.io) is a premier IT development company located in the financial capital of India, Mumbai. We specialize in delivering cutting-edge technology solutions and IT services, catering to businesses across various domains and helping them thrive in a digital landscape. Our services include custom software development, web and mobile application development, IT consulting, IT Outsourcing/Staffing, and Digital transformation.

In addition to our commercial endeavors, WOLFx Digital Agency is deeply committed to the open-source community. We actively contribute to open-source projects, sharing our expertise and innovations with the wider tech community.

## Contribution

We welcome contributions to improve and expand the capabilities of WOLFx Admin Configuration. Please feel free to submit issues, feature requests, or pull requests.

## License

This project is licensed under the MIT License. See the LICENSE file for details.
