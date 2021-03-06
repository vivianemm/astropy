.. include:: references.txt

.. _astropy-table:

*****************************
Data Tables (`astropy.table`)
*****************************

Introduction
============

`astropy.table` provides functionality for storing and manipulating
heterogeneous tables of data in a way that is familiar to `numpy` users.  A few
notable capabilities of this package are:

* Initialize a table from a wide variety of input data structures and types.
* Modify a table by adding or removing columns, changing column names,
  or adding new rows of data.
* Handle tables containing missing values.
* Include table and column metadata as flexible data structures.
* Specify a description, units and output formatting for columns.
* Interactively scroll through long tables similar to using ``more``.
* Create a new table by selecting rows or columns from a table.
* Perform :ref:`table_operations` like database joins, concatenation, and binning.
* Maintain a table index for fast retrieval of table items or ranges.
* Manipulate multidimensional columns.
* Handle non-native (mixin) column types within table.
* Methods for :ref:`read_write_tables` to files.
* Hooks for :ref:`subclassing_table` and its component classes.

Currently `astropy.table` is used when reading an ASCII table using
`astropy.io.ascii`.  Future releases of Astropy are expected to use
the |Table| class for other subpackages such as `astropy.io.votable` and `astropy.io.fits` .

.. Warning:: Astropy 2.0 introduces an API change that affects comparison of
   bytestring column elements in Python 3.  See
   :ref:`bytestring-columns-python-3` for details.

Getting Started
===============

The basic workflow for creating a table, accessing table elements,
and modifying the table is shown below.  These examples show a very simple
case, while the full `astropy.table` documentation is available from the
:ref:`using_astropy_table` section.

First create a simple table with three columns of data named ``a``, ``b``,
and ``c``.  These columns have integer, float, and string values respectively::

  >>> from astropy.table import Table
  >>> a = [1, 4, 5]
  >>> b = [2.0, 5.0, 8.2]
  >>> c = ['x', 'y', 'z']
  >>> t = Table([a, b, c], names=('a', 'b', 'c'), meta={'name': 'first table'})

If you have row-oriented input data such as a list of records, use the ``rows``
keyword.  In this example we also explicitly set the data types for each column::

  >>> data_rows = [(1, 2.0, 'x'),
  ...              (4, 5.0, 'y'),
  ...              (5, 8.2, 'z')]
  >>> t = Table(rows=data_rows, names=('a', 'b', 'c'), meta={'name': 'first table'},
  ...           dtype=('i4', 'f8', 'S1'))

There are a few ways to examine the table.  You can get detailed information
about the table values and column definitions as follows::

  >>> t  # doctest: +IGNORE_OUTPUT_3
  <Table length=3>
    a      b     c
  int32 float64 str1
  ----- ------- ----
      1     2.0    x
      4     5.0    y
      5     8.2    z

You can also assign a unit to the columns. If any column has a unit
assigned, all units would be shown as follows::

  >>> t['b'].unit = 's'
  >>> t  # doctest: +IGNORE_OUTPUT_3
  <Table length=3>
    a      b       c
           s
  int32 float64 str1
  ----- ------- ----
      1     2.0    x
      4     5.0    y
      5     8.2    z

Finally, you can get summary information about the table as follows::

  >>> t.info  # doctest: +IGNORE_OUTPUT_3
  <Table length=3>
  name  dtype  unit
  ---- ------- ----
     a   int32
     b float64    s
     c    str1

A column with a unit works with and can be easily converted to an
`~astropy.units.Quantity` object (but see :ref:`quantity_and_qtable` for
a way to natively use `~astropy.units.Quantity` objects in tables)::

  >>> t['b'].quantity  # doctest: +FLOAT_CMP
  <Quantity [2. , 5. , 8.2] s>
  >>> t['b'].to('min')  # doctest: +FLOAT_CMP
  <Quantity [0.03333333, 0.08333333, 0.13666667] min>

From within the IPython notebook, the table is displayed as a formatted HTML
table (details of how it appears can be changed by altering the
``astropy.table.default_notebook_table_class`` configuration item):

.. image:: table_repr_html.png

Or you can get a fancier notebook interface with in-browser search and sort
using `~astropy.table.Table.show_in_notebook`:

.. image:: table_show_in_nb.png

If you print the table (either from the notebook or in a text console session)
then a formatted version appears::

  >>> print(t)
   a   b   c
       s
  --- --- ---
    1 2.0   x
    4 5.0   y
    5 8.2   z

If you do not like the format of a particular column, you can change it::

  >>> t['b'].format = '7.3f'
  >>> print(t)
   a     b     c
         s
  --- ------- ---
    1   2.000   x
    4   5.000   y
    5   8.200   z

For a long table you can scroll up and down through the table one page at
time::

  >>> t.more()  # doctest: +SKIP

You can also display it as an HTML-formatted table in the browser::

  >>> t.show_in_browser()  # doctest: +SKIP

or as an interactive (searchable & sortable) javascript table::

  >>> t.show_in_browser(jsviewer=True)  # doctest: +SKIP

Now examine some high-level information about the table::

  >>> t.colnames
  ['a', 'b', 'c']
  >>> len(t)
  3
  >>> t.meta
  {'name': 'first table'}

Access the data by column or row using familiar `numpy` structured array syntax::

  >>> t['a']       # Column 'a'
  <Column name='a' dtype='int32' length=3>
  1
  4
  5

  >>> t['a'][1]    # Row 1 of column 'a'
  4

  >>> t[1]         # Row object for table row index=1 # doctest: +IGNORE_OUTPUT_3
  <Row index=1>
    a      b     c
           s
  int32 float64 str1
  ----- ------- ----
      4   5.000    y


  >>> t[1]['a']    # Column 'a' of row 1
  4

You can retrieve a subset of a table by rows (using a slice) or
columns (using column names), where the subset is returned as a new table::

  >>> print(t[0:2])      # Table object with rows 0 and 1
   a     b     c
         s
  --- ------- ---
    1   2.000   x
    4   5.000   y

  >>> print(t['a', 'c'])  # Table with cols 'a', 'c'
   a   c
  --- ---
    1   x
    4   y
    5   z

Modifying table values in place is flexible and works as one would expect::

  >>> t['a'][:] = [-1, -2, -3]    # Set all column values in place
  >>> t['a'][2] = 30              # Set row 2 of column 'a'
  >>> t[1] = (8, 9.0, "W")        # Set all row values
  >>> t[1]['b'] = -9              # Set column 'b' of row 1
  >>> t[0:2]['b'] = 100.0         # Set column 'b' of rows 0 and 1
  >>> print(t)
   a     b     c
         s
  --- ------- ---
   -1 100.000   x
    8 100.000   W
   30   8.200   z

Replace, add, remove, and rename columns with the following::

  >>> t['b'] = ['a', 'new', 'dtype']   # Replace column b (different from in place)
  >>> t['d'] = [1, 2, 3]               # Add column d
  >>> del t['c']                       # Delete column c
  >>> t.rename_column('a', 'A')        # Rename column a to A
  >>> t.colnames
  ['A', 'b', 'd']

Adding a new row of data to the table is as follows::

  >>> t.add_row([-8, -9, 10])
  >>> len(t)
  4

You can create a table with support for missing values, for example by setting
``masked=True``::

  >>> t = Table([a, b, c], names=('a', 'b', 'c'), masked=True, dtype=('i4', 'f8', 'S1'))
  >>> t['a'].mask = [True, True, False]
  >>> t  # doctest: +IGNORE_OUTPUT_3
  <Table masked=True length=3>
    a      b     c
  int32 float64 str1
  ----- ------- ----
     --     2.0    x
     --     5.0    y
      5     8.2    z

You can include certain object types like `~astropy.time.Time`,
`~astropy.coordinates.SkyCoord` or `~astropy.units.Quantity` in your table.
These "mixin" columns behave like a hybrid of a regular `~astropy.table.Column`
and the native object type (see :ref:`mixin_columns`).  For example::

  >>> from astropy.time import Time
  >>> from astropy.coordinates import SkyCoord
  >>> tm = Time(['2000:002', '2002:345'])
  >>> sc = SkyCoord([10, 20], [-45, +40], unit='deg')
  >>> t = Table([tm, sc], names=['time', 'skycoord'])
  >>> t
  <Table length=2>
           time          skycoord
                         deg,deg
          object          object
  --------------------- ----------
  2000:002:00:00:00.000 10.0,-45.0
  2002:345:00:00:00.000  20.0,40.0

The `~astropy.table.QTable` class is a variant of `~astropy.table.Table` in
which `~astropy.units.Quantity` are used natively, instead of being
converted to `~astropy.table.Column`. This means their units get taken into
account in numerical operations, etc. In this class `~astropy.table.Column`
is still used for all unit-less arrays (see :ref:`quantity_and_qtable`
for details)::

  >>> from astropy.table import QTable
  >>> import astropy.units as u
  >>> t = QTable()
  >>> t['dist'] = [1, 2] * u.m
  >>> t['velocity'] = [3, 4] * u.m / u.s
  >>> t['flag'] = [True, False]
  >>> t
  <QTable length=2>
    dist  velocity  flag
     m     m / s
  float64 float64   bool
  ------- -------- -----
      1.0      3.0  True
      2.0      4.0 False
  >>> t.info()
  <QTable length=2>
    name    dtype   unit  class
  -------- ------- ----- --------
      dist float64     m Quantity
  velocity float64 m / s Quantity
      flag    bool         Column

.. Note::

   The **only** difference between `~astropy.table.QTable` and
   `~astropy.table.Table` is the behavior when adding a column that has a
   specified unit.  With `~astropy.table.QTable` such a column is always
   converted to a `~astropy.units.Quantity` object before being added to the
   table.  Likewise if a unit is specified for an existing unit-less
   `~astropy.table.Column` in a `~astropy.table.QTable`, then the column is
   converted to `~astropy.units.Quantity`.

   The converse is that if one adds a `~astropy.units.Quantity` column to an
   ordinary `~astropy.table.Table` then it gets converted to an ordinary
   `~astropy.table.Column` with the corresponding ``unit`` attribute.

.. _using_astropy_table:

Using ``table``
===============

The details of using `astropy.table` are provided in the following sections:

Construct table
---------------

.. toctree::
   :maxdepth: 2

   construct_table.rst

Access table
---------------

.. toctree::
   :maxdepth: 2

   access_table.rst

Modify table
---------------

.. toctree::
   :maxdepth: 2

   modify_table.rst

Table operations
-----------------

.. toctree::
   :maxdepth: 2

   operations.rst

Indexing
--------

.. toctree::
   :maxdepth: 2

   indexing.rst

Masking
---------------

.. toctree::
   :maxdepth: 2

   masking.rst

I/O with tables
----------------

.. toctree::
   :maxdepth: 2

   io.rst
   pandas.rst

Mixin columns
----------------

.. toctree::
   :maxdepth: 2

   mixin_columns.rst

Implementation
----------------

.. toctree::
   :maxdepth: 2

   implementation_details.rst

Reference/API
=============

.. automodapi:: astropy.table
