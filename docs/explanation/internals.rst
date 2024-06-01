#############
CMS internals
#############

The following shall give an introduction to the internals of django
CMS. Reading this should help anyone who is interested in adding
functionality to the CMS or fixing bugs by modifiying the project's
code. It could also be helpful in gaining a deeper understanding of
the CMS for people wishing to extend the CMS through add-on
applications or through it's regular means of extension: custom
plugins, apphooks, wizards or menus.

The prerequisits for working on the django CMS code depend on the area
of interest. It can be either html, css, and javascript for the
frontend parts or django and python for the backend parts.

In order to understand the django CMS, you need to know that django
CMS is first and foremost a regular *django application*
[#multiple_apps]_. It is therefore structured like most other django
applications. This also means that customizing and extending it, can
be done by the regular means used in django.

Django CMS models can for example be searched and changed using the
regular django object relational mapper. There is usually no magic
preventing you from eg. creating your own objects like pages or
plugins and saving them to the database. Similarly, you can install or
extend existing templates or work with django admin classes.

.. warning:: People with little knowledge of internals of the CMS
             should use the :mod:`cms.api` interface instead of
             directly interfacing with Model classes or managers.

Building on the regular django concepts, the CMS introduces its own
mechanisms and concepts. The most important one is the structure
editor, the heart of the CMS, allowing the editor to create and modify
plugins in a page. The CMS also has a toolbar that can be customized
by the programmer by adding menu items or other user interface
elements. Another unique mechanism implemented by django CMS are
wizards, multi step processes helping editors to create
content. Lastly, the CMS implements two important ways to allow users
to create custom behavior: *custom plugins* and *app-hooks*.

Custom plugins can be used to define new types of content that editors
can add by themselves. This can be used for simple things like adding
a pre-defined logo to complex database backed data display widgets.

App-hooks allow you to seamlessly integrate other django apps into CMS
pages. A CMS project can thus be made of regular pages consisting of
plugins and of apphook pages that display other applications.

We will now approach the django CMS project from different
perspectives. We'll start off by looking at the main **concepts** used
by editors and the **model classes** implementing them. We can then
look at how the client (browser) interacts with the CMS for displaying
and for editing pages using the **structure editor**. Finally, we
finish by having a look at the **code layout**, showing where the most
important code can be found.

*******************
Concepts and models
*******************

To keep things simple, we will start by talking about the concepts
that an editor works with when using django CMS. Once the concepts
have been explained, we'll see how they are implemented in django CMS
using plain django models.

.. note:: The naming of these concepts might clash with the names of
          classes in django CMS. The former names refer to concepts
          that regular users of the CMS might talk about, the latter
          ones are used by django CMS programmers.

Page
====

Concept
-------

The central concept of most CMSes is the **page**. A page is a named
container for its content (plugins). Its main purpose is to be
rendered and displayed to the end user of the CMS, ie. the website
visitor. It may be displayed in a menu structure and has metadata such
as author name, creation datetime, slug (short name for URLs),
etc. Each time the user clicks on an internal link on a django CMS
website, a page will be requested from the server, rendered, sent to
the users browser and displayed there.

A page also has a template associated with it. This defines the html
code that will go around the actual content of the page like menus or
logos. Different pages can use different templates on the same django
CMS website.

Django CMS' pages are part of a *tree structure*. Each page can thus
have multiple *child* pages and zero or one *parent* pages. This allows
for pages to be part of a navigatable hierarchy that can be presented
to the user as a nested menu.

.. note:: The base django CMS does not define what exactly it means
          for a page to be the parent of another page. The usual
          interpretation is though that they shall be part of a
          hierarchical menu and that the lower level pages inherit
          some properties from the parents such as the template.

The system also comes with support for multiple languages and multiple
versions of the same page. A page can for example be requested in
mandarin or in spanish. Similarly, a page can exist in multiple
versions, allowing for elaborate editing schemes. Only one version of
a specific language and page is published and therefore visible to the
end user at the same time.

A django CMS page is therefore not only simple container for content
but more of a collection of multiple versions of content in different
languages.

Model implementation
--------------------

We just saw how things work on a conceptual level. In reality things
are a bit more complicated. Pages are represented by three model
classes working together. The :class:`Page` class is used to group
together different :class:`PageContent` objects which are referenced
by the actual content of pages. Each class:`PageContent` object
represents the translation into one language.

The tree structure is implemented using :class:`TreeNode` objects that
reference other :class:`TreeNode` objects through their `parent`
links. Each :class:`Page` object is linked to a :class:`TreeNode`
object which attaches it to the tree.

The following figure shows an example menu from a fictional website
for a music club. The first page is available in german and english
language while the others are only available in english. The
"Locations" page has two sub-pages.

The bottom part of the diagram shows the model objects representing
this structure in django CMS as explained above.

.. figure:: example_pages.drawio.svg
   :alt: Example layout of model objects representing some CMS pages.

   Top: Only three pages are shown, the rest are left out for
   simplicity. 

   Bottom: The :class:`TreeNode <cms.models.pagemodel.TreeNode>`
   objects are used to represent the relationship between pages. The
   pages represent the logical pages independent of the language. The
   :class:`PageContent <cms.models.pagecontent.PageContent>` objects
   represent the different language versions of each page.

Page models support some additional properties. The optional
`reverse_id` property is used to reference the page by a unique name
give to it by the editor. The `navigation_extenders` property can be
used to attach a named menu to a page. This only makes sense when
using the menu app described later. The `login_required` property
defines wether only authenticated useres can access a page or if it is
public. The `is_home` property can be true on only one page; this page
will be the default page when the site is first visited. The
`languages` property defines a list of all the languages the page has
translations for. The `is_page_type` defines wether this page is a
template page that can be used to create other pages. This can be
useful when the editor needs to create many similar pages starting
from a common state.

Finally, the `application_urls` and `application_namespace` properties
are used when the page is defined by an apphook, ie. it will display a
third party app instead of showing plugins.

Placeholder and Plugin
======================

Concept
-------

A **plugin** is the smallest element of content that the CMS deals
with directly. There can be many different types of plugins, each
serving a different purpose. A page's content is built out of plugins,
each rendering a different part of the final html.

Different plugin-types (title, text, image, ...) can save different
information within their instances. A header plugin instance might
save the text of the header element to be rendered and its level
[#header_level]_ for example. An image plugin, on the other hand,
needs to save a reference to the image to be displayed but might also
keep track of its size or wether a frame should be drawn around the
image.

Plugins are assembled in ordered tree structures. This means that they
can be put *one after the other* but they can also be placed *inside*
each others. Not all plugin-types support being a child of other
types. It would for example not make sense to allow putting a header
plugin into another header plugin. On the other hand, it would make
sense to put a header plugin into a column plugin.

It sometimes makes sense to insert plugins into different places in a
page's template. A classical example is a website where the main
content needs to be changed by editors and on the bottom of the page,
a footer needs to be editable as well. This type of page therefore
needs two separate places to put plugins. In django CMS these places
are called **placeholders**. It is therefore the placeholders that hold
the plugins, not the pages directly. The placeholders on the other
hand are held by the page [#placeholder_template_tag]_.

Model Implementation
--------------------

As mentionned before, Plugin objects belong to Placeholders, which in
turn are referenced from :class:`PageContent` objects. Different kind
of Plugin objects all inherit from the same :class:`CMSPlugin` class.

Each :class:`CMSPlugin` part of a placeholder part of a Placeholder,
is part of an ordered tree. This tree is defined using the `position`
[#unique]_ and the `parent` field. A placeholder having three plugins
in a flat list would be encoded using a `position` of 1, 2 and 3
respectively. The `parent` field of all the plugins would be `null`
since there is no nesting.

A structure with a text plugin inside a column plugin inside a row
plugin would be encoded by the `parent` of the text plugin pointing to
the column and the `parent` of the column plugin pointing to the row
plugin. The `position` fields would have the values 3, 2 and 1
respectively.

.. figure:: placeholder_plugins.drawio.svg
	    :alt: Placehoders and plugins example
		  
	    test

Menu
====

Concepts
--------

The django CMS allows you to group menu items in **ordered** tree
structures calle menus. These menus can be rendered in the website
using pre-existing django template tags.

Each menu item is composed of a title, unique id and a URL that it
refers to. It also links to the parent menu and child items.

By itself the django CMS menus are not connected to the concept of
pages. It is however very easy to generate a menu from a tree of
pages. In this case, each menu item represents a page and the page
hierarchy is re-traced by the menu's items. Each item's URL will also
point to the URL of the corresponding page.

The reason for this design is that menus can be generated by all kinds
of means, not only by page hierarchies. You could for example create a
menu whose items are the dates of the last seven days and that link to
filters on a blog for those days.

In order to flexibly display the menus, you can define modifier
classes that can change menus on the fly. One example is a modifier
that removes all items above a certain node in order to make the menu
more compact.

Model implementation
--------------------

FIXME

**************************
Client- Server interaction
**************************

FIXME

******************
Source code layout
******************

FIXME

.. rubric:: Footnotes

.. [#multiple_apps] Django CMS is actually composed of two
                    applications, the cms app which delivers most of
                    the functionality and the menus app which provides
                    support for building menus.
.. [#header_level] h1, h2, h3, etc
.. [#placeholder_template_tag] The the position of a placeholder and
                               that of its plugins, needs to be
                               defined in the template associated with
                               the page. This can be done by the means
                               of a special template tag.
.. [#unique] the position is unique for a placeholder and language
             combination
