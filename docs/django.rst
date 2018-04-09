.. _hypothesis-django:

===========================
Hypothesis for Django users
===========================

Hypothesis offers a number of features specific for Django testing, available
in the :mod:`hypothesis[django]` :doc:`extra </extras>`.  This is tested
against each supported series with mainstream or extended support -
if you're still getting security patches, you can test with Hypothesis.

Using it is quite straightforward: All you need to do is subclass
:class:`hypothesis.extra.django.TestCase` or
:class:`hypothesis.extra.django.TransactionTestCase`
and you can use :func:`@given <hypothesis.given>` as normal,
and the transactions will be per example
rather than per test function as they would be if you used :func:`@given <hypothesis.given>` with a normal
django test suite (this is important because your test function will be called
multiple times and you don't want them to interfere with each other). Test cases
on these classes that do not use
:func:`@given <hypothesis.given>` will be run as normal.

I strongly recommend not using
:class:`~hypothesis.extra.django.TransactionTestCase`
unless you really have to.
Because Hypothesis runs this in a loop the performance problems it normally has
are significantly exacerbated and your tests will be really slow.
If you are using :class:`~hypothesis.extra.django.TransactionTestCase`,
you may need to use ``@settings(suppress_health_check=[HealthCheck.too_slow])``
to avoid :doc:`errors due to slow example generation </healthchecks>`.

Having set up a test class, you can now pass :func:`@given <hypothesis.given>`
a strategy for Django models:

.. autofunction:: hypothesis.extra.django.models.models

For example, using the trivial django project I have for testing:

.. code-block:: python

    >>> from hypothesis.extra.django.models import models
    >>> from toystore.models import Customer
    >>> c = models(Customer).example()
    >>> c
    <Customer: Customer object>
    >>> c.email
    'jaime.urbina@gmail.com'
    >>> c.name
    '\U00109d3d\U000e07be\U000165f8\U0003fabf\U000c12cd\U000f1910\U00059f12\U000519b0\U0003fabf\U000f1910\U000423fb\U000423fb\U00059f12\U000e07be\U000c12cd\U000e07be\U000519b0\U000165f8\U0003fabf\U0007bc31'
    >>> c.age
    -873375803

Hypothesis has just created this with whatever the relevant type of data is.

Obviously the customer's age is implausible, which is only possible because
we have not used (eg) :class:`~django:django.core.validators.MinValueValidator`
to set the valid range for this field (or used a
:class:`~django:django.db.models.PositiveSmallIntegerField`, which would only
need a maximum value validator).

If you *do* have validators attached, Hypothesis will only generate examples
that pass validation.  Sometimes that will mean that we fail a
:class:`~hypothesis.HealthCheck` because of the filtering, so let's explicitly
pass a strategy to skip validation at the strategy level:

.. note::
    Inference from validators will be much more powerful when :issue:`1116`
    is implemented, but there will always be some edge cases that require you
    to pass an explicit strategy.

.. code-block:: python

    >>> from hypothesis.strategies import integers
    >>> c = models(Customer, age=integers(min_value=0, max_value=120)).example()
    >>> c
    <Customer: Customer object>
    >>> c.age
    5

---------------
Tips and tricks
---------------

Custom field types
==================

If you have a custom Django field type you can register it with Hypothesis's
model deriving functionality by registering a default strategy for it:

.. code-block:: python

    >>> from toystore.models import CustomishField, Customish
    >>> models(Customish).example()
    hypothesis.errors.InvalidArgument: Missing arguments for mandatory field
        customish for model Customish
    >>> from hypothesis.extra.django.models import add_default_field_mapping
    >>> from hypothesis.strategies import just
    >>> add_default_field_mapping(CustomishField, just("hi"))
    >>> x = models(Customish).example()
    >>> x.customish
    'hi'

Note that this mapping is on exact type. Subtypes will not inherit it.


Generating child models
=======================

For the moment there's no explicit support in hypothesis-django for generating
dependent models. i.e. a Company model will generate no Shops. However if you
want to generate some dependent models as well, you can emulate this by using
the *flatmap* function as follows:

.. code:: python

  from hypothesis.strategies import lists, just

  def generate_with_shops(company):
    return lists(models(Shop, company=just(company))).map(lambda _: company)

  company_with_shops_strategy = models(Company).flatmap(generate_with_shops)

Lets unpack what this is doing:

The way flatmap works is that we draw a value from the original strategy, then
apply a function to it which gives us a new strategy. We then draw a value from
*that* strategy. So in this case we're first drawing a company, and then we're
drawing a list of shops belonging to that company: The *just* strategy is a
strategy such that drawing it always produces the individual value, so
``models(Shop, company=just(company))`` is a strategy that generates a Shop belonging
to the original company.

So the following code would give us a list of shops all belonging to the same
company:

.. code:: python

  models(Company).flatmap(lambda c: lists(models(Shop, company=just(c))))

The only difference from this and the above is that we want the company, not
the shops. This is where the inner map comes in. We build the list of shops
and then throw it away, instead returning the company we started for. This
works because the models that Hypothesis generates are saved in the database,
so we're essentially running the inner strategy purely for the side effect of
creating those children in the database.


Using default field values
==========================

Hypothesis ignores field defaults and always tries to generate values, even if
it doesn't know how to. You can tell it to use the default value for a field
instead of generating one by passing ``fieldname=default_value`` to
``models()``:

.. code:: python

    >>> from toystore.models import DefaultCustomish
    >>> models(DefaultCustomish).example()
    hypothesis.errors.InvalidArgument: Missing arguments for mandatory field
        customish for model DefaultCustomish
    >>> from hypothesis.extra.django.models import default_value
    >>> x = models(DefaultCustomish, customish=default_value).example()
    >>> x.customish
    'b'
