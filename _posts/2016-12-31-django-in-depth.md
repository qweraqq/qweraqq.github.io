---
layout: post
title: "Django in depth"
date: 2016-12-31 00:00:00 +0800
author: xiangxiang
categories: python django web
tags: [python django]
---
学习Django in Depth - James Bennett - PyCon 2015 [ppt](http://media.b-list.org/s/django-in-depth-2015.pdf)

## Part 1: ORM
```
Model
  |
Manager
  |
QuerySet
  |
Query
  |
SQLComplier
  |
Database backend 
```
### Model
+ Model metaclasses
	+ first, learn [python metaclasses](https://realpython.com/python-metaclasses/)
	+ django.db.models.base.ModelBase
	+ Does the basic setup of the model class
	+ Handles `Meta` declaration, sets up default manager if needed, adds in model-specific exception subclasses, etc (Model Meta options)[https://docs.djangoproject.com/en/2.2/ref/models/options/]
	+ ModelBase then loops through the `attribute dictionary` (type('name', tuple_base_class, attr_dict)) of the new class, and calls  `contribute_to_class()` on anything which defines it
+ Model fields
	+ Subclasses of django.db.models.Field
	+ data types
		+ `get_internal_type()` can return a built-in field type if similar enough to that type
		+ `db_type()` can return a custom database-level type for the field
	+ value conversion
		+ `to_python()` converts from DB-level type to correct Python type
		+ `value_to_string()` converts to string for serialization purposes
		+ Multiple methods for preparing values for various kinds of DB-level use (querying, storage, etc.)
	+ misc
		+ `formfield()` returns a default form field to use for this field
		+ `value_from_object()` takes an instance of the model, returns the value for this field on that instance
		+ `deconstruct()` is used for data migrations (i.e., what to pass to __init__() to reconstruct this value later)

### Manager
+ The high-level interface
+ Attach directly to a model class (and know which model they’re attached to, via self.model)
+ Create and return a QuerySet for you
+ Expose most of the methods of QuerySet
+ Options
	+ Don’t specify one at all — Django will create a default one and name it objects
	+ If you specify one, Django doesn’t create the default objects manager
	+ One model class can have multiple managers
	+ First one defined becomes the default (accessible via _default_manager)
+ How it works
	+ Manager’s `get_queryset()` method returns a QuerySet for the model
	+ Almost everything else just proxies through to that
	+ Except `raw()`, which instantiates a `RawQuerySet`
+ Caution
	+ Overriding `get_queryset()` will affect all queries made through that manager
	+ **Usually best to keep a vanilla manager around so you can access everything normally**

### QuerySet (django/db/models/query.py)
+ Wraps a Query instance in a nice API
+ Actually implements the high-level query methods you use
+ Acts as a container-ish object for the results (IT’S A CONTAINER)
	+ Each instance of QuerySet has an internal results cache
	+ Iterating will pull from the results cache if populated rather than re-executing the query
	+ IMPORTANT: Calling a QuerySet method will usually clone the pre-existing QuerySet, apply the change, and return the new instance (doing a new query)
		+ A performance trade-off
		+ Except for iteration, length/existence checks, which can re-use the existing QuerySet instance’s results cache without doing a new query
```python
>>> my_queryset = SomeModel.objects.all()
>>> my_queryset[5].some_attribute # clone & new query
3
>>> my_queryset[5].some_attribute = 2 # clone & new query
>>> my_queryset[5].save() # clone & new query
>>> my_queryset[5].some_attribute # clone & new query
3
>>> # WAT
```
+ LAZINESS: Most of the time, does nothing except twiddle its (and the wrapped Query’s) attributes in response to method calls, returning the modified QuerySet object
+ How to force a quert
	+ Call a method that doesn’t return a QuerySet
	+ Call a method which requires actually querying the database to populate results, check existence of results, check the number of results, etc.
	+ Including special Python methods
		+ Implements __iter__() to be iterable
		+ __len__(), __bool__(), __nonzero__()
		+ __getitem__ (for slicing/indexing)
+ ONE SPECIAL NOTE
	+ `__repr__()` will perform a query, and will add a `LIMIT 21` to it
	+ Saves you from yourself: accidentally repr()-ing a QuerySet with a million results in it is bad for your computer/server

### Query (django/db/models/sql/query.py)
In simple terms, a Query is a tree-like data structure, and turning it into SQL involves going through its various nodes, calling the as_sql() method of each one to get its output.
+ Data structure and methods representing a database query
+ Lives in django/db/models/sql/
+ Two flavors: Query (normal ORM operations) and RawQuery (for raw())
+ Basic structure
	+ Attributes representing query parts
	+ select, tables, where, group_by, order_by, aggregates, etc
	+ The as_sql() method pulls it all together and makes nice (we hope) SQL out of it

**The bridge (multi database backend support)**
1. Query.get_compiler() returns a SQLCompiler instance for that Query
2. Which calls the compiler() method of DatabaseOperations, passing the name of the desired compiler
3. Which in turn looks at DatabaseOperations.compiler_module to find location of SQLCompiler classes for current backend

### SQLComplier
+ Turns a Django Query instance into SQL for your database
+ Subclasses for non-SELECT queries
	+ SQLInsertCompiler for INSERT
	+ SQLDeleteCompiler for DELETE, etc

### Database backend 
+ boundary between Django and the Database driver modules
+ 实际中最相关的是db的settings
+ 组成
	+ DatabaseWrapper: subclasses BaseDatabaseWrapper from django.db.backends.base
		+ Carries basic information about the SQL dialect it speaks (column types, lookup and pattern operators,etc.)
		+ Control transaction
	+ DatabaseOperations
		+ Type casts/ value extraction
		+ flushes and sequence resets (about sequence)[https://stackoverflow.com/questions/1649102/what-is-a-sequence-database-when-would-we-need-it]
	+ DatabaseFeatures
		+ What does this database support
		+ Whether casts are needs/ some types exist
		+ SELECT/FOR/UPDATE support
		+ how NULL ording works
		+ etc.
	+ DatabaseCreation
		+ Quirks of creating the database/tables
		+ Mostly now deals with the variations in index creation/removal
		+ Make migrations/Migrate
	+ DatabaseIntrospection
		+ `inspectdb` management command
		+ reverse engineering: database tables -> Django models
	+ DatabaseSchemaEditor
		+ Used in migrations
		+ Knows DB-specific things about modifying the schema
	+ DatabaseClient
		+ `dbshell` management command
		+ Knows how to open interactive console session to the DB

### Flavors of inheritance
1. Abstract parents
+ `abstract = True` in the `Meta` declaration
+ Subclasses, if not abstract, generate a table with both their own fields and the abstract parent’s
+ Subclasses can subclass and/or override abstract parent’s Meta

2. Multi-table
+ No special syntax: just subclass the other model
+ Can’t directly subclass parent Meta, but can override by re-declaring options
+ Can add new fields, managers, etc.
+ Subclass has implicit `OneToOneField` to parent

3. Proxy models
+ Subclass parent, declare `proxy = True` in Meta
+ Will reuse parent’s table and fields; only allowed changes are at the Python level
+ Proxy subclass can define additional methods, manager, etc
+ Queries return instances of the model queried; query a proxy, get instances of the proxy

### Unmanaged models
+ Set `managed = False` in `Meta`
+ Django will not do any DB management for this model: won’t create tables, won’t track migrations, won’t auto-add a primary key if missing, etc
+ Can declare fields, define Meta options, define methods normally aside from that

```
Unmanaged models are mostly useful for wrapping a pre-existing DB table, or DB view,
that you don’t want Django to try to mess with.
While they can be used to emulate some aspects of model inheritance, 
declaring a proxy subclass of a model is almost always a better way of doing that.
```


## Part 2: Forms
### Widgets
+ Low-level operations
+ In: Know how to pull their data out of a submission
	+ When data is submitted, a widget’s value_from_datadict() method pulls out that widget’s value
+ Out: Know how to render appropriate HTML/ Can bundle arbitrary media (CSS, JavaScript) to
include and use when rendering
	+ When displaying a form, a widget’s render() method is responsible for generating that widget’s HTML
+ MultiWidget: Special class that wraps multiple other widgets
	+ Useful for things like split date/time
	+ decompress() method unpacks a single value into multiple

### Fields
+ Represent data type and validation constraints
+ Have widgets associated with them for rendering
+ Validataion data (Call field’s `clean()` method) --- from HTTP request
	1. This calls field’s `to_python()` method to convert value to correct **Python data type**
	2. Then calls field’s `validate()` method to run field’s own *built-in* validation **more specific format, such as email/url**
	3. Then calls field’s run_validators() method to run any additional validators supplied by end user
	4. clean() either returns a valid value, or raises ValidationError

FIELDS AND WIDGETS
+ Each field technically has two widgets: second one is used when it’s a hidden field
+ widget_attrs() method gets passed the widget instance, and can return a dictionary of HTML attributes to pass to it

### Forms (Tie fields/widgets all together in a neat package)
+ Provide the high-level interface you’ll actually use
+ You can use it directly if you want: django.forms.BaseForm
+ But probably not a good idea unless you’re constructing form classes dynamically at runtime
+ Forms use metaclass (which is a bad idea)
	+ The main thing to know about is that a form class ends up with two field-related `attributes` (type('name', tuple_base_class, attr_dict)):
	1. base_fields: the default set of fields for all instances of that form class
	2. fields: the set of fields for a **specific instance** of the form class
+ BUILD A FORM
```python
base_fields = {
	'name': forms.CharField(max_length=255),
	'email': forms.EmailField(),
	'message': forms.CharField(widget=forms.Textarea),
	}
ContactForm = type('ContactForm',
					(forms.BaseForm,),
					{'base_fields': base_fields})
```
```python
class ContactForm(forms.Form):
	name = forms.CharField(max_length=255)
	email = forms.EmailField()
	message = forms.CharField(widget=forms.Textarea)
```
+ Working with data
	+ Instantiating a form with data will cause it to wrap its fields in instances of `django.forms.forms.BoundField`
	+ Each BoundField has a field instance, a reference to the form it’s in and the field’s name within that form
	+ BoundField instances are what you get when you **iterate over the form**
	+ Validation (Call the form’s is_valid() method)
		1. That applies field-level validation
			1. For each field, call its widget’s value_from_datadict() to get the value for the field
			2. Call field’s clean() method
			3. Call any `clean_<fieldname>()` method on the form (if field clean() ran without errors))
			4. Errors go into form’s errors dict, keyed by field name
		2. Then form-level validation
			1. Form’s clean() method
			2. Happens after field validation, so cleaned_data may or may not be populated depending on prior errors
			3. Errors raised here end up in error dict under the key __all__, or by calling non_field_errors()
		3. Then sets up either cleaned_data or errors attribute
+ Displaying a form
	+ Default representation of a form is as an HTML table
	+ Also built in: unordered list, or paragraphs
	+ None of these output the containing `<form></form>` or the submit elements
	+ `_html_output()` method can be useful for building your own custom form HTML output

### ModelForm: Model <-> form conversion
+ Introspects a model class and makes a form out of it
+ Basic structure (declarative class plus inner options class) is similar to models
+ Uses model meta API to get lists of fields, etc.
+ Override/configure things by setting options in `Meta`
+ Getting form fields and values
	+ Calls the `formfield()` method of each field in the model
	+ Can be overridden by defining the method `formfield_callback()` on the form
	+ Each field’s value_from_object() method called to get the value of that field on that model instance
+ Saving
	+ django.forms.models.construct_instance() builds an instance of the model with form data filled in
	+ django.forms.models.save_instance() actually (if commit=True) saves the instance to the DB
	+ Saving of many-to-many relations is deferred until after the instance itself has been saved; with commit=False, you have to manually do it

### Media support (form media)
+ Forms and widgets support inner class named `Media`
+ Which in turn supports values css and js, listing CSS and JavaScript assets to include when rendering the form/widget
+ css is a dict where keys are CSS media types; js is just a list
+ Media objects are combinable, and results only include one of each asset

```
Actually, any class can support media definitions; 
widgets and forms are just the ones that support them by default.
To add media to another class, have that class use the metaclass django.forms.widgets.MediaDefiningClass. 
It will parse an inner Media declaration and turn it into a media property with the same behaviors
(i.e., combinable, prints as HTML, etc.) as on forms.
```

## Part 3: Template 
[Django templates](https://docs.djangoproject.com/en/2.2/topics/templates/)

Django ships built-in backends for its own template system, creatively called the Django template language (DTL), 
and for the popular alternative [Jinja2](https://palletsprojects.com/p/jinja/)

### Engine
+ Subclass django.template.backends.base.BaseEngine
+ Must implement get_template(), can optionally implement other methods
+ Typically will define a wrapper object around the template for consistent API

### Loader
+ Do the hard work of actually finding the template you asked for
+ Required method: `load_template_source()`,taking template name and optional directories to
look in
+ Should return a 2-tuple of template contents and template location

### Template objects
+ Instantiate with template source
+ Implement render() method receiving optional context (dictionary) and optional request (HttpRequest object), returning string

### Tag and filter libraries

### Context

### The Django template language
+ Key classes
	+ django/template/base.py
	+ Template is the high-level representation
	+ Lexer and Parser actually turn the template source into something usable
	+ Token represents bits of the template during parsing
	+ Node and NodeList are the structures which make up a compiled Template

+ Template lexing
	+ Instantiate with template source, then call `tokenize()`
	+ Splits the template source using a **regex** which recognizes start/end syntax of tags and variables
	+ Creates and returns a list of tokens based on the resulting substrings
		+ Tokens that come in four flavors: Text, Var, Block, Comment
		+ Tokens that get the text of some piece of template syntax, minus the start/end constructs like \{\% … \%\} or \{\{ … \}\}


+ Template parsing
	+ Instantiate Parser with list of tokens from Lexer, then call `parse()`
	+ Each tag and filter (built-in or custom) gets registered with Parser
	+ Each tag provides a compilation function, which Parser will call to produce a Node
	+ Variables get automatically translated into VariableNode, plain text into TextNode

+ Tag compilation
	+ Each tag must provide this
	+ It gets access to the token representing the tag, and to the parser
	+ Can just be simple and instantiate/return a Node subclass
	+ Or can use the parser to do more complex things like parsing ahead to an end tag, doing things with all the nodes in between, etc.

+ Nodes
	+ Can be an individual Node, or a NodeList
	+ Defining characteristic is the render() method which takes a Context as argument and returns a string
	```
	The full process transforms the string source of the template into a NodeList, 
	with one Node for each tag, each variable and each bit of plain text.
	Then, rendering the template simply involves having that NodeList iterate over and `render()` its constituent nodes, 
	concatenating the results into a single string which is the output.
	```

+ Variables
	+ Basic representation is a VariableNode
	+ Filters are represented by FilterExpression which parses out filter names, looks up the correct functions and applies them
	+ `render()` consists of resolving the variable in the context, applying the filters, returning the result

+ Template context
	+ Lives in django/template/context.py
	+ Behaves like a dictionary
	+ Is actually a stack of dictionaries, supporting push/pop and fall-through lookups
	+ First dictionary in the stack to return a value wins

    ```text
    The stack implementation of Context is crucial,
    since it allows tags to easily set groups of variables by pushing a new dictionary on top,
    then clean up after themselves by popping that dictionary back off the stack.
    
    Several of the more interesting built-in tags usethis pattern.
    
    However, be aware of the performance implications: 
    just as nested namespaces in Python code will slow down name lookups, 
    so too lots of nested dictionaries on the context stack will slow down variable resolution.
    ```

+ RenderContext
	+ Attached to every Context: is a thread-safety tool for simultaneous renderings of the same template instance
	+ Each call to render() pushes its own dictionary on top, pops when done rendering, and only the topmost dictionary is searched during name resolution in RenderContext
	+ Tags with thread-safety issues can store their state in the RenderContext and know they’ll get the right state


## Part 4: Request/Response processing
### Entry point
+ Request handler, in django/core/handlers
+ WSGIHandler is the only one supported now 
+ Implements a WSGI application [PEP 333](https://www.python.org/dev/peps/pep-0333/)
+ ASGI in django 3

### Handler lifecycle
+ Sets up middleware
+ Sends request_started signal
+ Initializes HttpRequest object
+ Handler calls its own get_response() method
	+ Apply request middleware
	+ Resolve URL
		+ High-level
			+ django.core.urls.RegexURLResolver
			+ Takes URLconf import path, and optional prefix (for include())
			+ Will import the URLconf and get its list of patterns
			+ Implements the resolve() method which returns the actual view and arguments to use
		+ URL patterns
			+ Instances of RegexURLPattern
			+ Stores regex, callback (the view), and optional name and other arguments
			+ Implements a resolve() method to indicate whether a given URL matches it
		+ Matches
			+ Returned in the form of a ResolverMatch object
			+ Unpacks into view, positional arguments, keyword arguments
			+ Has other information attached, including optional namespace, app name, etc
		+ URL resolution
			+ RegexURLResolver iterates over the supplied patterns, popping prefixes as necessary (for include())
			+ Keeps a list of patterns it tried and didn’t get a match on
			+ Returns the first time it gets a ResolverMatch from calling resolve() on a RegexURLPattern
			+ Or raises Resolver404 if no match, includes list of failed patterns for debugging
	+ Apply view middleware
	+ Call view
	+ Apply response middleware
	+ Return response
	```
	The bulk of get_response() is actually error handling. 
	It wraps everything in try/except, implements the exception handling for failed URL resolution and errors in middleware or views, 
	and also wraps the view in a transaction which can be rolled back in case of error.
	This resolve URL process is potentially nested many times;
	each URLconf you include() from your root URLconf will have its own associated RegexURLResolver (prefixed).
	There is short-circuit logic to check that the prefix matches before proceeding through the patterns, 
	to avoid pointlessly doing match attempts that are guaranteed to fail.
	Under the hood, the RegexURLResolver of the root URLconf is given a prefix of "^/"
	```
+ ERROR HANDLING (**不要泄露错误信息**)
	+ Handler’s get_exception_response() method is invoked
	+ First tries to look up a handler for the error type in the root URLconf (handler404, handler500, etc.) and call and return that response
	+ If all else fails, handler’s handle_uncaught_exception() is called
	+ **Django never attempts to catch SystemExit**
```
The error handling is robust in the sense that it can keep information from leaking, 
but not in the sense of always returning something friendly to the end user.
If the initial handling of an error fails, Django promotes it to an error 500 and tries to invoke the handler500 view. 
If attempts to handle the exception keep raising new ones, Django gives up and lets the exception propagate out.
For this reason, it’s important to have your handler500 be as bulletproof as possible, or just use Django’s default implementation.
```
+ Transforms HttpResponse into appropriate outgoing format, and returns that
+ REQUEST AND RESPONSE
	+ HttpRequest is technically handler-specific, but there’s only one handler now
	+ HttpResponse comes in multiple flavors depending on status code
	+ Built-in: 301, 302, 400, 403, 404, 405, 410, 500
	+ Plus JsonResponse for JSON output
+ Views
	+ Must meet three requirements to be a Django view
		1. callable
		2. Accepts an HttpRequest as first positional argument
		3. Returns an HttpResponse or raises an exception
	+ 主要需要关注:functions or classes实现view
		+ Most end-user-written views are likely to be functions, at least at first
		+ Class-based views are more useful when reusing/customizing behavior
	+ class-based generic views
		+ Inheritance diagram is complex
		+ Actual use is not
		+ Most of the base classes and mixins exist to let functionality be composed à la carte in the classes end users will actually be working with
		+ The basics
			+ self.request is the current HttpRequest
			+ Dispatch is done by dispatch() method, based on request method, to methods of the correct name: get(), post(), etc.
			+ You probably want TemplateResponseMixin somewhere in your inheritance chain if it’s not already, for easy template loading/rendering
			+ Call as_view() [class method] when putting it in a URLconf, which ensures the dispatch() method will be what’s called