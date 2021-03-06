V2: new tox multi-dimensional, platform-specific configuration
--------------------------------------------------------------------

.. note::

   This is a draft document sketching a to-be-done implementation.
   It does not fully specify each change yet but should give a good
   idea of where things are heading.  For feedback, mail the
   testing-in-python mailing list or open a pull request at
   https://github.com/tox-dev/tox.

**Abstract**: Adding multi-dimensional configuration, platform-specification
and multiple installers to tox.ini.

**Target audience**: Developers using or wanting to use tox for testing
their python projects.

Issues with current tox (1.4) configuration
------------------------------------------------

tox is used as a tool for creating and managing virtualenv environments
and running tests in them. As of tox-1.4 there are some issues frequently
coming up with its configuration language:

- there is no way to instruct tox to parametrize testenv specifications
  other than to list all combinations by specifying a ``[testenv:...]``
  section for each combination. Examples of real life situations
  arising from this:

  * https://github.com/encode/django-rest-framework/blob/b001a146d73348af18cfc4c943d87f2f389349c9/tox.ini

  * https://bitbucket.org/tabo/django-treebeard/src/93b579395a9c/tox.ini

- there is no way to have platform specific settings other than to
  define specific testenvs and invoke tox with a platform-specific
  testenv list.

- there is no way to specify the platforms against which a project
  shall successfully run.

- tox always uses pip for installing packages currently.  This has
  several issues:

  - no way to check if installing via easy_install works
  - no installs of packages with compiled c-extensions (win32 standard)


Goals, resolving those issues
------------------------------------

This document discusses a possible solution for each of these issues,
namely these goals:

- allow to more easily define and run dependency/interpreter variants
  with testenvs
- allow platform-specific settings
- allow to specify platforms against which tests should run
- allow to run installer-variants (easy_install or pip, xxx)
- try to mimick/re-use bash-style syntax to ease learning curve.


Example: Generating and selecting variants
----------------------------------------------

Suppose you want to test your package against python3.6, python2.7 and on the
windows and linux platforms.  Today you would have to
write down 2*2 = 4 ``[testenv:*]`` sections and then instruct
tox to run a specific list of environments on each platform.

With tox-1.X you can directly specify combinations::

    # combination syntax gives 2 * 2 = 4 testenv names
    #
    envlist = {py27,py36}-{win,linux}

    [testenv]
    deps = pytest
    platform =
           win: windows
           linux: linux
    basepython =
           py27: python2.7
           py36: python3.6
    commands = pytest

Let's go through this step by step::

    envlist = {py27,py36}-{windows,linux}

This is bash-style syntax and will create ``2*2=4`` environment names
like this::

    py27-windows
    py27-linux
    py36-windows
    py36-linux

Our ``[testenv]`` uses a new templating style for the ``platform`` definition::

    platform=
           windows: windows
           linux: linux

These two conditional settings will lead to either ``windows`` or
``linux`` as the platform string.  When the test environment is run,
its platform string needs to be contained in the string returned
from ``platform.platform()``. Otherwise the environment will be skipped.

The next configuration item in the ``testenv`` section deals with
the python interpreter::

    basepython =
           py27: python2.7
           py36: python3.6

This defines a python executable, depending on if ``py36`` or ``py27``
appears in the environment name.

The last config item is simply the invocation of the test runner::

    commands = pytest

Nothing special here :)

.. note::

    tox provides good defaults for platform and basepython
    settings, so the above ini-file can be further reduced::

        [tox]
        envlist = {py27,py36}-{win,linux}

        [testenv]
        deps = pytest
        commands = pytest

    Voila, this multi-dimensional ``tox.ini`` configuration
    defines 2*2=4 environments.


The new "platform" setting
--------------------------------------

A testenv can define a new ``platform`` setting.  If its value
is not contained in the string obtained from calling
``sys.platform`` the environment will be skipped.

Expanding the ``envlist`` setting
----------------------------------------------------------

The new ``envlist`` setting allows to use ``{}`` bash-style
expressions.  XXX explanation or pointer to bash-docs

Templating based on environment names
-------------------------------------------------

For a given environment name, all lines in a testenv section which
start with "NAME: ..." will be checked for being part in the environment
name.  If they are part of it, the remainder will be the new line.
If they are not part of it, the whole line will be left out.
Parts of an environment name are obtained by ``-``-splitting it.

Variant specification with [variant:VARNAME]

Showing all expanded sections
-------------------------------

To help with understanding how the variants will produce section values,
you can ask tox to show their expansion with a new option::

    $ tox -l [XXX output omitted for now]

Making sure your package installs with easy_install
------------------------------------------------------

The new "installer" testenv setting allows to specify the tool for
installation in a given test environment::

    [testenv]
    installer =
        easy: easy_install
        pip: pip

If you want to have your package installed with both easy_install
and pip, you can list them in your envlist like this::

    [tox]
    envlist = py[27,35,36]-django[13,14]-[easy,pip]

If no installer is specified, ``pip`` will be used.

Default settings related to environment names/variants
---------------------------------------------------------------

tox comes with predefined settings for certain variants, namely:

* ``{easy,pip}`` use easy_install or pip respectively
* ``{py27,py34,py35,py36,pypy19]`` use the respective
  pythonNN or PyPy interpreter
* ``{win32,linux,darwin}`` defines the according ``platform``.

You can use those in your “envlist” specification
without the need to define them yourself.


Use more bash-style syntax
--------------------------------------

tox leverages bash-style syntax if you specify mintoxversion = 1.4:

- $VARNAME or ${...} syntax instead of the older {} substitution.
- XXX go through config.rst and see how it would need to be changed


Transforming the examples: django-rest
------------------------------------------------

The original `django-rest-framework tox.ini
<https://github.com/encode/django-rest-framework/blob/b001a146d73348af18cfc4c943d87f2f389349c9/tox.ini>`_
file has 159 lines and a lot of repetition, the new one would have ``20+``
lines and almost no repetition::

     [tox]
     envlist = {py27,py35,py36}-{django12,django13}{,-example}

     [testenv]
     deps=
         coverage==3.4
         unittest-xml-reporting==1.2
         Pyyaml==3.10
         django12: django==1.2.4
         django13: django==1.3.1
         # some more deps for running examples
         example: wsgiref==0.1.2
         example: Pygments==1.4
         example: httplib2==0.6.0
         example: Markdown==2.0.3

     commands =
        !example: python setup.py test
        example: python examples/runtests.py


Note that ``{,-example}`` in the envlist denotes two values, an empty
one and a ``example`` one.  The empty value means that there are no specific
settings and thus no need to define a variant name.

Transforming the examples: django-treebeard
------------------------------------------------

Another `tox.ini
<https://bitbucket.org/tabo/django-treebeard/raw/93b579395a9c/tox.ini>`_
has 233 lines and runs tests against multiple Postgres and Mysql
engines.  It also performs backend-specific test commands, passing
different command line options to the test script.  With the new tox-1.X
we not only can do the same with 32 non-repetive configuration lines but
we also produce 36 specific testenvs with specific dependencies and test
commands::

    [tox]
    envlist =
     {py27,py34,py35,py36}-{django11,django12,django13}-{nodb,pg,mysql}, docs

    [testenv:docs]
    changedir = docs
    deps =
        Sphinx
        Django
    commands =
        make clean
        make html

     [testenv]
     deps=
           coverage
           pysqlite
           django11: django==1.1.4
           django12: django==1.2.7
           django13: django==1.3.1
           django14: django==1.4
           nodb: pysqlite
           pg: psycopg2
           mysql: MySQL-python

     commands =
         nodb: {envpython} runtests.py {posargs}
         pg: {envpython} runtests.py {posargs} \
                         --DATABASE_ENGINE=postgresql_psycopg2 \
                         --DATABASE_USER=postgres {posargs}
         mysql: {envpython} runtests.py --DATABASE_ENGINE=mysql \
                                        --DATABASE_USER=root {posargs}
