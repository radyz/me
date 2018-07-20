description: Improving DRF page number paginator performance for large models

# DRF page number paginator performance

There's often the case where we encounter applications which require paginated
lists of data and of course being a very common pattern, Django and DRF
already provide solutions for dealing with this. We would simply hook one of
[DRF's](http://www.django-rest-framework.org/api-guide/pagination/)
built in paginator classes and be done with it. However as our application
grows we start adding new properties, foreign relationships by the dozens to
our model and all of a sudden our response time starts to degrade[^1].

## So, what happened?

Without getting into much detail, Django's orm is doing its job of building
a database query behind the scenes for you. However in doing so, it usually
fetches each and every property of your model and all related models which are
being selected in that query, which translated to the actual database call
causes a lot of overhead.


## Ehm... how can I fix this?

Well as it turns out you could use
[Django's](https://docs.djangoproject.com/en/2.0/ref/models/querysets/#django.db.models.query.QuerySet.only)
built in queryset methods `.only` or `.defer` to specify only the properties of
your model and related models to avoid this problem. You could end up with
something like this:

```python
return queryset.only(
    'property_1',
    'property_2',
    'related_model_property_1',
    'related_model_property_2', 
    ...)
```

Well you get the idea... However the above solution will get ugly very fast when
we are talking about 10+ properties, it's just not going to scale for our
specific use case.

## Then there's nothing we can do

Well, actually there's one solution which has proven very effective to me.
Let's recap for a second, we are suffering due to having many columns being
selected in our query but we **need them** and having to specify them all for each
one of our models which have these issues is not going to be pretty.

:bulb: So how about if we could fetch our models without columns first, build a
query with their pk value only and then make our final query to fetch our models
fully?.

## Show me the code already!!!

Below is my current solution[^2] to this situation. I've added comments to each
relevant step, the rest is just Django's code which needs to be there.

```python
from django.core.paginator import Paginator as BasePaginator
from django.utils.functional import cached_property

from rest_framework.pagination import PageNumberPagination as BasePaginator
from rest_framework.viewsets import ModelViewSet


# We need to create our own paginator subclass using Django's base
class EagerPaginator(BasePaginator):
    # Let's hijack the query building process to create our own
    def page(self, number):
        number = self.validate_number(number)
        bottom = (number - 1) * self.per_page
        top = bottom + self.per_page
        if top + self.orphans >= self.count:
            top = self.count

        # We grab only the 'pk' property for our model. In case there's any
        # annotation set, we need to grab those as they may be used with filters
        properties = ['pk'] + [annotation for annotation
                               in self.object_list.query.annotations.keys()]

        # So now that we have the minimum amount of properties to be selected,
        # we execute the query with bottom:top limits with only those properties.
        matches = list(self.object_list.values(*properties)[bottom:top])
        # Now we have all our matches filtered, ordered and annotated, however
        # this time there's no overhead in our database

        # We have already executed filtering and ordering in the previous step
        # so there's no need to have these anymore and so we clear them out
        self.object_list.query.where = WhereNode(connector=OR)
        self.object_list.query.clear_ordering(force_empty=True)

        page = []
        # Now we proceed to query by 'pk' only with our matches to fetch our
        # models fully with all their properties
        items = list(self.object_list.filter(
            pk__in=[match.get('pk') for match in matches]))

        # Since our query by 'pk' does not guarantee order, we order the
        # results back as we know them from matches
        for match in matches:
            page.append(next(
                item for item in items if item.pk == match.get('pk')))

        return self._get_page(page, number, self)

    @cached_property
    def count(self):
        # In the rare case we have defined an extra clause in our queryset we
        # must remove it during count and then re-attach it again
        try:
            extra = self.object_list_query.extra.copy()
            self.object_list.query.extra.clear()

            count = self.object_list.count()
            self.object_list.query.extra.update(extra)

            return count
        except (AttributeError, TypeError):
            return len(self.object_list)


# Now we proceed to create our DRF subclass
class EagerPageNumberPagination(PageNumberPagination):
    django_paginator_class = EagerPaginator


# Profit
class CustomModelViewSet(ModelViewSet):
    pagination_class = EagerPageNumberPagination
```

And there you go :beers:

## Downsides

Each request will now issue 3 database queries to your database as opposed of
the 2 it normally does. However the performance gains out weight this stat [^1]

[^1]: Always measure before making any assumptions.
[Django's debug toolbar](https://django-debug-toolbar.readthedocs.io/en/stable/)
is an excellent tool for this purpose
[^2]: Tested against Django 1.10.x, 1.11.x and DRF 3.6.x
