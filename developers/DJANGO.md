---
layout: default
title: Django
parent: Programming
nav_order: 12
---

# Django

**Purpose**

To share things we've found to solve common issues with Django and save lots of time.

**Scope**

Covers numerous specific issues and makes overall recommendations.

## Client Cache Management

Include version string in static assets paths loaded from the client.

    <script src="script.js?v={{ settings.VERSION }}"></script>

In `settings.py` set `VERSION=os.system('git rev-parse HEAD')`

This is a wonderful catchall to prevent client cache issues that waste
so much time.

## Use LTS Django

Use the long term support version of Django, ideally the latest one available at any given time.

## Read 2 Scoops of Django

[This](https://mongard.s3.ir-thr-at1.arvanstorage.com/Two%20Scoops%20of%20Django%203.x%20by%20Daniel%20Audrey%20Feldroy%20(z-lib.org).pdf) book contains a lot of great practices, which we *almost* universally agree with. In particular:

  - Fat models, skinny views. Put more logic and code in Model methods, if it's relevant to that model specifically. Avoid large amounts of code in views.

## Learn Database Basics

Django's ORM is amazing most of the time. But it's very easy to do the wrong thing. [This](https://thebuild.com/presentations/unbreaking-django.pdf) is a great deck that covers the many caveats of the ORM and how to avoid them, recommended for advanced Django users in particular since it assumes some basic knowledge. If it's too dense, read 2 scoops (above) first.

## Patterns

### Imports

Certain files in Django should only import from other certain types of files. 

For example, `urls.py` should really only import from `views.py` (and `utils.py`).

![import flow](/assets/images/django_import_flow.jpg)

### Models

 - Keep models *normalized*: no duplicate data or extra foreign keys.
 - Schema changes should be reviewed by a senior dev, as any bad designs here will cascade to other layers.

### Views

 - We generally use function based views (FBV) instead of class based views (CBV) at Countable.
 - When using Django Rest Framework (DRF), this is an exception and we prefer CBV. Please do use DRF for substantial rest API work.
 - Keep views small. It's better to put logic in Models or other shared modules outside the views.
 - Avoid unnecessary nesting.

```
    #unnecessary nesting:
    def view(request):
        if 'x' in request.GET:
            if (another check):
                return HttpResponse(...)
    # better
    def view(request):
        if 'x' not in request.GET:
            return Http400(...)
        if not another check:
            return Http400(...)
        return HttpResponse(...)
```

### Guidelines

 - Use <span class="title-ref">help\_text</span> in your models because it helps both devs and users in the `/admin` . 
 
 ie) `color = models.CharField(help_text="Your primary brand color as a hex code, ie) #FF0000")`

### Timezone Management

  - Timezones are a complicated aspect to manage. After working on this on some large projects, here are recommendations:
      - You should usually avoid DateField, and it's better to store DateTimeField because then we can switch timezones later on if needed. (Most events do have time at which they occurred, and that should be stored.)
      - Store Datetimes in the database in UTC.
      - Only convert to a specific timezone at the View/Template layer (when rendering or collectin information from the user).
      - Note Python has several timezone libraries that can sometimes [conflict or have performance issues](https://blog.ganssle.io/articles/2018/03/pytz-fastest-footgun.html). The fast and correct way to set a timezone of a datetime object is `dt_object.replace(tzinfo=ZoneInfo("America/Pacific"))` .
      - Do not use the default server timezone or hardcode a timezone at any point. You should use UTC, or get the specific timezone from the domain objects (frontend UI should specify it)
### Managing Environments

  - Django should have SMTP creds, and should set the ADMINS to the back-end developer that maintains that project. Or, use sentry\_sdk with [sentry.countable.ca](sentry.countable.ca)
  - In prod and staging environments, set DEBUG=False, so the ADMINS get emails with any stack traces.


### Performance

To enable SQL logging in Docker logs, edit the following line in `postgresql.conf` and restart your Docker container:
```
log_statement = 'all'                  # none, ddl, mod, all
```

The most common performance issue in Django is when we put queries inside of iteration. Queries should always happen OUTSIDE of for loops, for example.

```
bad:
for x in Modelname.objects.filter(...):
    print(x.foreign_key)
bad:
for x in Modelname.objects.filter(...):
    print(ForeignModel.objects.get(modelname=x))
good:
for x in Modelname.objects.filter(...).select_related('foreign_key')
    print(x.foreign_key.foreign_key)
```

`.select_related()` should be used for foreign keys, and `.prefetch_related()` should be used to many-to-many relationships or reverse foreign keys.

Note: if you are using Django Rest Framework views and serializers, you may need to override functions such as `.list()` or `.retrieve()`, with the optimized query and manually pass it into the serializer; e.g.
```
def retrieve(self, request, *args, **kwargs):
    form = self.request.clinic.diagnosticform_set.prefetch_related('diagnosticquestion_set', 'diagnosticquestion_set__answer_to_question', 'diagnosticquestion_set__workflow').get(id=self.kwargs['pk'])
    serializer = DiagnosticFormDetailSerializer(form)
    return Response(serializer.data)
```

Further reading: [DB Optimization in Django](https://docs.djangoproject.com/en/4.1/topics/db/optimization/)

### Templates

Django templates should generally be replaced by a modern, API-driven front-end. However, when using templates keep the following in mind:

If you're passing `|safe` (i.e. HTML or JSON) strings directly in an HTML template, you should use the `backtick` string templates, because they support newlines that would otherwise break the page rendering where "regular quoted strings" don't.

```
const customDisabledPrivatePatientSignupMessage = `{{ request.clinic.custom_disabled_private_patient_signup_message|safe }}`;
```

Also, when referencing static files, be sure to manage the cache proactively using a token. A Git Hash works well:

{% raw %}
```
<script src="{% static 'asset.js' %}?ver={{ request.git_hash }}"></script>
```
{% endraw %}

### Error Reporting and Sensitive Data

When running in prod with error reporting for exceptions (e.g. Sentry or email logs), use Django's decorators `@sensitive_variables()` and `@sensitive_post_parameters` to avoid passing sensitive data to error handling systems. **Make sure to put the `@sensitive_variables()` decorator first so that it works through the other decorators.** [Read more here](https://docs.djangoproject.com/en/4.0/howto/error-reporting/#filtering-sensitive-information).

```
@sensitive_variables('patient_id', 'cell_phone')  # @sensitive_variables should always be the first decorator
@my_other_decorator
def my_function():
    # ...
```

### Common Gotchas

**Model fields that are JSONFields with default values**

When making a JSONField model fields with default values, do not return a plain list (`[]`) or dict (`{}`), because this will cause that instance of list/dict to be shared across all model instances. Instead, return `list()` or `dict()`.

```
def get_default_json_list():
    return []
workflows = models.JSONField(default=get_default_json_list)  # bad


def get_default_json_list():
    return list()
workflows = models.JSONField(default=get_default_json_list)  # correct
```

**Concurrency safety**

When saving a model, make sure you load a fresh copy of the object right before saving to avoid a race condition. This will avoid very difficult-to-track bugs that are caused when another function/person modifies the object before your function saves the (now stale) object and overwrites their changes. To be safer, load the clinic fresh, AND it's best to call save with the `update_fields` [parameter](https://docs.djangoproject.com/en/2.0/ref/models/instances/#specifying-which-fields-to-save).

```
clinic.long_running_process_that_updates_data()
...
# Clinic might be stale now.
fresh_clinic = Clinic.objects.get(pk=clinic.id)
fresh_clinic.some_attribute = "new value"
fresh_clinic.save(update_fields=["some_attribute"])
```

This greatly reduces the risk and is good enough for many cases, but if you require true concurrency safety you must implement a lock mechanism (pessimistic) or check the model's modified_date has not changed (optimistic).

**Querying models in apps with multitenancy**

When querying models in apps with multitenancy where most of the models have a foreign key back to a main or identifying model, it's better to access these secondary models by reverse foreign key rather than querying the object directly. For example:

```
class Clinic(models.Model):
    # the main model
    # fields...

class Workflow(models.Model):
    # secondary model, specific to a clinic
    clinic = models.ForeignKey(Clinic)
    # fields...

clinic = Clinic.objects.get(...)

# bad
Workflow.objects.get(clinic=clinic, ...)

# preferred
clinic.workflow_set.get(...)
```
The second method is preferred because it avoids an easy mistake (forgetting to add the `clinic=clinic` filter) that causes serious issues (mixing data between customers).

Note: if you're confused about the reverse foreign key lookup, the [Django docs](https://docs.djangoproject.com/en/dev/topics/db/queries/#following-relationships-backward) explain in more detail how managers work. If a custom `related_name` has not been set, the default format should be `[modelname]_set`. You can also open a Django shell; autocomplete will show you what the available reverse lookups are:
```
>>> clinic.specialization [press tab to autocomplete]
clinic.specializationpolicies_set(  clinic.specializations(
```
