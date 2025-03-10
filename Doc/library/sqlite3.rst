:mod:`sqlite3` --- DB-API 2.0 interface for SQLite databases
============================================================

.. module:: sqlite3
   :synopsis: A DB-API 2.0 implementation using SQLite 3.x.

.. sectionauthor:: Gerhard Häring <gh@ghaering.de>

**Source code:** :source:`Lib/sqlite3/`

.. Make sure we always doctest the tutorial with an empty database.

.. testsetup::

   import sqlite3
   src = sqlite3.connect(":memory:", isolation_level=None)
   dst = sqlite3.connect("tutorial.db", isolation_level=None)
   src.backup(dst)
   del src, dst

.. _sqlite3-intro:

SQLite is a C library that provides a lightweight disk-based database that
doesn't require a separate server process and allows accessing the database
using a nonstandard variant of the SQL query language. Some applications can use
SQLite for internal data storage.  It's also possible to prototype an
application using SQLite and then port the code to a larger database such as
PostgreSQL or Oracle.

The :mod:`!sqlite3` module was written by Gerhard Häring.  It provides an SQL interface
compliant with the DB-API 2.0 specification described by :pep:`249`, and
requires SQLite 3.7.15 or newer.

This document includes four main sections:

* :ref:`sqlite3-tutorial` teaches how to use the :mod:`!sqlite3` module.
* :ref:`sqlite3-reference` describes the classes and functions this module
  defines.
* :ref:`sqlite3-howtos` details how to handle specific tasks.
* :ref:`sqlite3-explanation` provides in-depth background on
  transaction control.

.. seealso::

   https://www.sqlite.org
      The SQLite web page; the documentation describes the syntax and the
      available data types for the supported SQL dialect.

   https://www.w3schools.com/sql/
      Tutorial, reference and examples for learning SQL syntax.

   :pep:`249` - Database API Specification 2.0
      PEP written by Marc-André Lemburg.


.. We use the following practises for SQL code:
   - UPPERCASE for keywords
   - snake_case for schema
   - single quotes for string literals
   - singular for table names
   - if needed, use double quotes for table and column names

.. _sqlite3-tutorial:

Tutorial
--------

In this tutorial, you will create a database of Monty Python movies
using basic :mod:`!sqlite3` functionality.
It assumes a fundamental understanding of database concepts,
including `cursors`_ and `transactions`_.

First, we need to create a new database and open
a database connection to allow :mod:`!sqlite3` to work with it.
Call :func:`sqlite3.connect` to to create a connection to
the database :file:`tutorial.db` in the current working directory,
implicitly creating it if it does not exist:

.. testcode::

   import sqlite3
   con = sqlite3.connect("tutorial.db")

The returned :class:`Connection` object ``con``
represents the connection to the on-disk database.

In order to execute SQL statements and fetch results from SQL queries,
we will need to use a database cursor.
Call :meth:`con.cursor() <Connection.cursor>` to create the :class:`Cursor`:

.. testcode::

   cur = con.cursor()

Now that we've got a database connection and a cursor,
we can create a database table ``movie`` with columns for title,
release year, and review score.
For simplicity, we can just use column names in the table declaration --
thanks to the `flexible typing`_ feature of SQLite,
specifying the data types is optional.
Execute the ``CREATE TABLE`` statement
by calling :meth:`cur.execute(...) <Cursor.execute>`:

.. testcode::

   cur.execute("CREATE TABLE movie(title, year, score)")

.. Ideally, we'd use sqlite_schema instead of sqlite_master below,
   but SQLite versions older than 3.33.0 do not recognise that variant.

We can verify that the new table has been created by querying
the ``sqlite_master`` table built-in to SQLite,
which should now contain an entry for the ``movie`` table definition
(see `The Schema Table`_ for details).
Execute that query by calling :meth:`cur.execute(...) <Cursor.execute>`,
assign the result to ``res``,
and call :meth:`res.fetchone() <Cursor.fetchone>` to fetch the resulting row:

.. doctest::

   >>> res = cur.execute("SELECT name FROM sqlite_master")
   >>> res.fetchone()
   ('movie',)

We can see that the table has been created,
as the query returns a :class:`tuple` containing the table's name.
If we query ``sqlite_master`` for a non-existent table ``spam``,
:meth:`!res.fetchone()` will return ``None``:

.. doctest::

   >>> res = cur.execute("SELECT name FROM sqlite_master WHERE name='spam'")
   >>> res.fetchone() is None
   True

Now, add two rows of data supplied as SQL literals
by executing an ``INSERT`` statement,
once again by calling :meth:`cur.execute(...) <Cursor.execute>`:

.. testcode::

   cur.execute("""
       INSERT INTO movie VALUES
           ('Monty Python and the Holy Grail', 1975, 8.2),
           ('And Now for Something Completely Different', 1971, 7.5)
   """)

The ``INSERT`` statement implicitly opens a transaction,
which needs to be committed before changes are saved in the database
(see :ref:`sqlite3-controlling-transactions` for details).
Call :meth:`con.commit() <Connection.commit>` on the connection object
to commit the transaction:

.. testcode::

   con.commit()

We can verify that the data was inserted correctly
by executing a ``SELECT`` query.
Use the now-familiar :meth:`cur.execute(...) <Cursor.execute>` to
assign the result to ``res``,
and call :meth:`res.fetchall() <Cursor.fetchall>` to return all resulting rows:

.. doctest::

   >>> res = cur.execute("SELECT score FROM movie")
   >>> res.fetchall()
   [(8.2,), (7.5,)]

The result is a :class:`list` of two :class:`!tuple`\s, one per row,
each containing that row's ``score`` value.

Now, insert three more rows by calling
:meth:`cur.executemany(...) <Cursor.executemany>`:

.. testcode::

   data = [
       ("Monty Python Live at the Hollywood Bowl", 1982, 7.9),
       ("Monty Python's The Meaning of Life", 1983, 7.5),
       ("Monty Python's Life of Brian", 1979, 8.0),
   ]
   cur.executemany("INSERT INTO movie VALUES(?, ?, ?)", data)
   con.commit()  # Remember to commit the transaction after executing INSERT.

Notice that ``?`` placeholders are used to bind ``data`` to the query.
Always use placeholders instead of :ref:`string formatting <tut-formatting>`
to bind Python values to SQL statements,
to avoid `SQL injection attacks`_
(see :ref:`sqlite3-placeholders` for more details).

We can verify that the new rows were inserted
by executing a ``SELECT`` query,
this time iterating over the results of the query:

.. doctest::

   >>> for row in cur.execute("SELECT year, title FROM movie ORDER BY year"):
   ...     print(row)
   (1971, 'And Now for Something Completely Different')
   (1975, 'Monty Python and the Holy Grail')
   (1979, "Monty Python's Life of Brian")
   (1982, 'Monty Python Live at the Hollywood Bowl')
   (1983, "Monty Python's The Meaning of Life")

Each row is a two-item :class:`tuple` of ``(year, title)``,
matching the columns selected in the query.

Finally, verify that the database has been written to disk
by calling :meth:`con.close() <Connection.close>`
to close the existing connection, opening a new one,
creating a new cursor, then querying the database:

.. doctest::

   >>> con.close()
   >>> new_con = sqlite3.connect("tutorial.db")
   >>> new_cur = new_con.cursor()
   >>> res = new_cur.execute("SELECT title, year FROM movie ORDER BY score DESC")
   >>> title, year = res.fetchone()
   >>> print(f'The highest scoring Monty Python movie is {title!r}, released in {year}')
   The highest scoring Monty Python movie is 'Monty Python and the Holy Grail', released in 1975

You've now created an SQLite database using the :mod:`!sqlite3` module,
inserted data and retrieved values from it in multiple ways.

.. _SQL injection attacks: https://en.wikipedia.org/wiki/SQL_injection
.. _The Schema Table: https://www.sqlite.org/schematab.html
.. _cursors: https://en.wikipedia.org/wiki/Cursor_(databases)
.. _flexible typing: https://www.sqlite.org/flextypegood.html
.. _sqlite_master: https://www.sqlite.org/schematab.html
.. _transactions: https://en.wikipedia.org/wiki/Database_transaction

.. seealso::

   * :ref:`sqlite3-howtos` for further reading:

      * :ref:`sqlite3-placeholders`
      * :ref:`sqlite3-adapters`
      * :ref:`sqlite3-converters`
      * :ref:`sqlite3-connection-context-manager`

   * :ref:`sqlite3-explanation` for in-depth background on transaction control.

.. _sqlite3-reference:

Reference
---------

.. We keep the old sqlite3-module-contents ref to prevent breaking links.
.. _sqlite3-module-contents:

.. _sqlite3-module-functions:

Module functions
^^^^^^^^^^^^^^^^

.. function:: connect(database, timeout=5.0, detect_types=0, \
                      isolation_level="DEFERRED", check_same_thread=True, \
                      factory=sqlite3.Connection, cached_statements=128, \
                      uri=False)

   Open a connection to an SQLite database.

   :param database:
       The path to the database file to be opened.
       Pass ``":memory:"`` to open a connection to a database that is
       in RAM instead of on disk.
   :type database: :term:`path-like object`

   :param float timeout:
       How many seconds the connection should wait before raising
       an exception, if the database is locked by another connection.
       If another connection opens a transaction to modify the database,
       it will be locked until that transaction is committed.
       Default five seconds.

   :param int detect_types:
       Control whether and how data types not
       :ref:`natively supported by SQLite <sqlite3-types>`
       are looked up to be converted to Python types,
       using the converters registered with :func:`register_converter`.
       Set it to any combination (using ``|``, bitwise or) of
       :const:`PARSE_DECLTYPES` and :const:`PARSE_COLNAMES`
       to enable this.
       Column names takes precedence over declared types if both flags are set.
       Types cannot be detected for generated fields (for example ``max(data)``),
       even when the *detect_types* parameter is set; :class:`str` will be
       returned instead.
       By default (``0``), type detection is disabled.

   :param isolation_level:
       The :attr:`~Connection.isolation_level` of the connection,
       controlling whether and how transactions are implicitly opened.
       Can be ``"DEFERRED"`` (default), ``"EXCLUSIVE"`` or ``"IMMEDIATE"``;
       or ``None`` to disable opening transactions implicitly.
       See :ref:`sqlite3-controlling-transactions` for more.
   :type isolation_level: str | None

   :param bool check_same_thread:
       If ``True`` (default), only the creating thread may use the connection.
       If ``False``, the connection may be shared across multiple threads;
       if so, write operations should be serialized by the user to avoid data
       corruption.

   :param Connection factory:
       A custom subclass of :class:`Connection` to create the connection with,
       if not the default :class:`Connection` class.

   :param int cached_statements:
       The number of statements that :mod:`!sqlite3`
       should internally cache for this connection, to avoid parsing overhead.
       By default, 128 statements.

   :param bool uri:
       If set to ``True``, *database* is interpreted as a
       :abbr:`URI (Uniform Resource Identifier)` with a file path
       and an optional query string.
       The scheme part *must* be ``"file:"``,
       and the path can be relative or absolute.
       The query string allows passing parameters to SQLite,
       enabling various :ref:`sqlite3-uri-tricks`.

   :rtype: Connection

   .. audit-event:: sqlite3.connect database sqlite3.connect
   .. audit-event:: sqlite3.connect/handle connection_handle sqlite3.connect

   .. versionadded:: 3.4
      The *uri* parameter.

   .. versionchanged:: 3.7
      *database* can now also be a :term:`path-like object`, not only a string.

   .. versionadded:: 3.10
      The ``sqlite3.connect/handle`` auditing event.

.. function:: complete_statement(statement)

   Return ``True`` if the string *statement* appears to contain
   one or more complete SQL statements.
   No syntactic verification or parsing of any kind is performed,
   other than checking that there are no unclosed string literals
   and the statement is terminated by a semicolon.

   For example:

   .. doctest::

      >>> sqlite3.complete_statement("SELECT foo FROM bar;")
      True
      >>> sqlite3.complete_statement("SELECT foo")
      False

   This function may be useful during command-line input
   to determine if the entered text seems to form a complete SQL statement,
   or if additional input is needed before calling :meth:`~Cursor.execute`.

   See :func:`!runsource` in :source:`Lib/sqlite3/__main__.py`
   for real-world use.

.. function:: enable_callback_tracebacks(flag, /)

   Enable or disable callback tracebacks.
   By default you will not get any tracebacks in user-defined functions,
   aggregates, converters, authorizer callbacks etc. If you want to debug them,
   you can call this function with *flag* set to ``True``. Afterwards, you
   will get tracebacks from callbacks on :data:`sys.stderr`. Use ``False``
   to disable the feature again.

   Register an :func:`unraisable hook handler <sys.unraisablehook>` for an
   improved debug experience:

   .. testsetup:: sqlite3.trace

      import sqlite3

   .. doctest:: sqlite3.trace

      >>> sqlite3.enable_callback_tracebacks(True)
      >>> con = sqlite3.connect(":memory:")
      >>> def evil_trace(stmt):
      ...     5/0
      >>> con.set_trace_callback(evil_trace)
      >>> def debug(unraisable):
      ...     print(f"{unraisable.exc_value!r} in callback {unraisable.object.__name__}")
      ...     print(f"Error message: {unraisable.err_msg}")
      >>> import sys
      >>> sys.unraisablehook = debug
      >>> cur = con.execute("SELECT 1")
      ZeroDivisionError('division by zero') in callback evil_trace
      Error message: None

.. function:: register_adapter(type, adapter, /)

   Register an *adapter* callable to adapt the Python type *type* into an
   SQLite type.
   The adapter is called with a Python object of type *type* as its sole
   argument, and must return a value of a
   :ref:`type that SQLite natively understands <sqlite3-types>`.

.. function:: register_converter(typename, converter, /)

   Register the *converter* callable to convert SQLite objects of type
   *typename* into a Python object of a specific type.
   The converter is invoked for all SQLite values of type *typename*;
   it is passed a :class:`bytes` object and should return an object of the
   desired Python type.
   Consult the parameter *detect_types* of
   :func:`connect` for information regarding how type detection works.

   Note: *typename* and the name of the type in your query are matched
   case-insensitively.


.. _sqlite3-module-constants:

Module constants
^^^^^^^^^^^^^^^^

.. data:: PARSE_COLNAMES

   Pass this flag value to the *detect_types* parameter of
   :func:`connect` to look up a converter function by
   using the type name, parsed from the query column name,
   as the converter dictionary key.
   The type name must be wrapped in square brackets (``[]``).

   .. code-block:: sql

      SELECT p as "p [point]" FROM test;  ! will look up converter "point"

   This flag may be combined with :const:`PARSE_DECLTYPES` using the ``|``
   (bitwise or) operator.

.. data:: PARSE_DECLTYPES

   Pass this flag value to the *detect_types* parameter of
   :func:`connect` to look up a converter function using
   the declared types for each column.
   The types are declared when the database table is created.
   :mod:`!sqlite3` will look up a converter function using the first word of the
   declared type as the converter dictionary key.
   For example:

   .. code-block:: sql

      CREATE TABLE test(
         i integer primary key,  ! will look up a converter named "integer"
         p point,                ! will look up a converter named "point"
         n number(10)            ! will look up a converter named "number"
       )

   This flag may be combined with :const:`PARSE_COLNAMES` using the ``|``
   (bitwise or) operator.

.. data:: SQLITE_OK
          SQLITE_DENY
          SQLITE_IGNORE

   Flags that should be returned by the *authorizer_callback* callable
   passed to :meth:`Connection.set_authorizer`, to indicate whether:

   * Access is allowed (:const:`!SQLITE_OK`),
   * The SQL statement should be aborted with an error (:const:`!SQLITE_DENY`)
   * The column should be treated as a ``NULL`` value (:const:`!SQLITE_IGNORE`)

.. data:: apilevel

   String constant stating the supported DB-API level. Required by the DB-API.
   Hard-coded to ``"2.0"``.

.. data:: paramstyle

   String constant stating the type of parameter marker formatting expected by
   the :mod:`!sqlite3` module. Required by the DB-API. Hard-coded to
   ``"qmark"``.

   .. note::

      The :mod:`!sqlite3` module supports both ``qmark`` and ``numeric`` DB-API
      parameter styles, because that is what the underlying SQLite library
      supports. However, the DB-API does not allow multiple values for
      the ``paramstyle`` attribute.

.. data:: sqlite_version

   Version number of the runtime SQLite library as a :class:`string <str>`.

.. data:: sqlite_version_info

   Version number of the runtime SQLite library as a :class:`tuple` of
   :class:`integers <int>`.

.. data:: threadsafety

   Integer constant required by the DB-API 2.0, stating the level of thread
   safety the :mod:`!sqlite3` module supports. This attribute is set based on
   the default `threading mode <https://sqlite.org/threadsafe.html>`_ the
   underlying SQLite library is compiled with. The SQLite threading modes are:

     1. **Single-thread**: In this mode, all mutexes are disabled and SQLite is
        unsafe to use in more than a single thread at once.
     2. **Multi-thread**: In this mode, SQLite can be safely used by multiple
        threads provided that no single database connection is used
        simultaneously in two or more threads.
     3. **Serialized**: In serialized mode, SQLite can be safely used by
        multiple threads with no restriction.

   The mappings from SQLite threading modes to DB-API 2.0 threadsafety levels
   are as follows:

   +------------------+-----------------+----------------------+-------------------------------+
   | SQLite threading | `threadsafety`_ | `SQLITE_THREADSAFE`_ | DB-API 2.0 meaning            |
   | mode             |                 |                      |                               |
   +==================+=================+======================+===============================+
   | single-thread    | 0               | 0                    | Threads may not share the     |
   |                  |                 |                      | module                        |
   +------------------+-----------------+----------------------+-------------------------------+
   | multi-thread     | 1               | 2                    | Threads may share the module, |
   |                  |                 |                      | but not connections           |
   +------------------+-----------------+----------------------+-------------------------------+
   | serialized       | 3               | 1                    | Threads may share the module, |
   |                  |                 |                      | connections and cursors       |
   +------------------+-----------------+----------------------+-------------------------------+

   .. _threadsafety: https://peps.python.org/pep-0249/#threadsafety
   .. _SQLITE_THREADSAFE: https://sqlite.org/compile.html#threadsafe

   .. versionchanged:: 3.11
      Set *threadsafety* dynamically instead of hard-coding it to ``1``.

.. data:: version

   Version number of this module as a :class:`string <str>`.
   This is not the version of the SQLite library.

   .. deprecated-removed:: 3.12 3.14
      This constant used to reflect the version number of the ``pysqlite``
      package, a third-party library which used to upstream changes to
      :mod:`!sqlite3`. Today, it carries no meaning or practical value.

.. data:: version_info

   Version number of this module as a :class:`tuple` of :class:`integers <int>`.
   This is not the version of the SQLite library.

   .. deprecated-removed:: 3.12 3.14
      This constant used to reflect the version number of the ``pysqlite``
      package, a third-party library which used to upstream changes to
      :mod:`!sqlite3`. Today, it carries no meaning or practical value.


.. _sqlite3-connection-objects:

Connection objects
^^^^^^^^^^^^^^^^^^

.. class:: Connection

   Each open SQLite database is represented by a ``Connection`` object,
   which is created using :func:`sqlite3.connect`.
   Their main purpose is creating :class:`Cursor` objects,
   and :ref:`sqlite3-controlling-transactions`.

   .. seealso::

      * :ref:`sqlite3-connection-shortcuts`
      * :ref:`sqlite3-connection-context-manager`

   An SQLite database connection has the following attributes and methods:

   .. attribute:: isolation_level

      This attribute controls the :ref:`transaction handling
      <sqlite3-controlling-transactions>` performed by :mod:`!sqlite3`.
      If set to ``None``, transactions are never implicitly opened.
      If set to one of ``"DEFERRED"``, ``"IMMEDIATE"``, or ``"EXCLUSIVE"``,
      corresponding to the underlying `SQLite transaction behaviour`_,
      implicit :ref:`transaction management
      <sqlite3-controlling-transactions>` is performed.

      If not overridden by the *isolation_level* parameter of :func:`connect`,
      the default is ``""``, which is an alias for ``"DEFERRED"``.

   .. attribute:: in_transaction

      This read-only attribute corresponds to the low-level SQLite
      `autocommit mode`_.

      ``True`` if a transaction is active (there are uncommitted changes),
      ``False`` otherwise.

      .. versionadded:: 3.2

   .. attribute:: row_factory

      A callable that accepts two arguments,
      a :class:`Cursor` object and the raw row results as a :class:`tuple`,
      and returns a custom object representing an SQLite row.

      Example:

      .. testcode::

         def dict_factory(cursor, row):
             d = {}
             for idx, col in enumerate(cursor.description):
                 d[col[0]] = row[idx]
             return d

         con = sqlite3.connect(":memory:")
         con.row_factory = dict_factory
         cur = con.execute("SELECT 1 AS a")
         print(cur.fetchone()["a"])

         con.close()

      .. testoutput::
         :hide:

         1

      If returning a tuple doesn't suffice and you want name-based access to
      columns, you should consider setting :attr:`row_factory` to the
      highly optimized :class:`sqlite3.Row` type. :class:`Row` provides both
      index-based and case-insensitive name-based access to columns with almost no
      memory overhead. It will probably be better than your own custom
      dictionary-based approach or even a db_row based solution.

      .. XXX what's a db_row-based solution?

   .. attribute:: text_factory

      A callable that accepts a :class:`bytes` parameter and returns a text
      representation of it.
      The callable is invoked for SQLite values with the ``TEXT`` data type.
      By default, this attribute is set to :class:`str`.
      If you want to return ``bytes`` instead, set *text_factory* to ``bytes``.

      Example:

      .. testcode::

         con = sqlite3.connect(":memory:")
         cur = con.cursor()

         AUSTRIA = "Österreich"

         # by default, rows are returned as str
         cur.execute("SELECT ?", (AUSTRIA,))
         row = cur.fetchone()
         assert row[0] == AUSTRIA

         # but we can make sqlite3 always return bytestrings ...
         con.text_factory = bytes
         cur.execute("SELECT ?", (AUSTRIA,))
         row = cur.fetchone()
         assert type(row[0]) is bytes
         # the bytestrings will be encoded in UTF-8, unless you stored garbage in the
         # database ...
         assert row[0] == AUSTRIA.encode("utf-8")

         # we can also implement a custom text_factory ...
         # here we implement one that appends "foo" to all strings
         con.text_factory = lambda x: x.decode("utf-8") + "foo"
         cur.execute("SELECT ?", ("bar",))
         row = cur.fetchone()
         assert row[0] == "barfoo"

         con.close()

   .. attribute:: total_changes

      Return the total number of database rows that have been modified, inserted, or
      deleted since the database connection was opened.


   .. method:: cursor(factory=Cursor)

      Create and return a :class:`Cursor` object.
      The cursor method accepts a single optional parameter *factory*. If
      supplied, this must be a callable returning an instance of :class:`Cursor`
      or its subclasses.

   .. method:: blobopen(table, column, row, /, \*, readonly=False, name="main")

      Open a :class:`Blob` handle to an existing
      :abbr:`BLOB (Binary Large OBject)`.

      :param str table:
          The name of the table where the blob is located.

      :param str column:
          The name of the column where the blob is located.

      :param str row:
          The name of the row where the blob is located.

      :param bool readonly:
          Set to ``True`` if the blob should be opened without write
          permissions.
          Defaults to ``False``.

      :param str name:
          The name of the database where the blob is located.
          Defaults to ``"main"``.

      :raises OperationalError:
          When trying to open a blob in a ``WITHOUT ROWID`` table.

      :rtype: Blob

      .. note::

         The blob size cannot be changed using the :class:`Blob` class.
         Use the SQL function ``zeroblob`` to create a blob with a fixed size.

      .. versionadded:: 3.11

   .. method:: commit()

      Commit any pending transaction to the database.
      If there is no open transaction, this method is a no-op.

   .. method:: rollback()

      Roll back to the start of any pending transaction.
      If there is no open transaction, this method is a no-op.

   .. method:: close()

      Close the database connection.
      Any pending transaction is not committed implicitly;
      make sure to :meth:`commit` before closing
      to avoid losing pending changes.

   .. method:: execute(sql, parameters=(), /)

      Create a new :class:`Cursor` object and call
      :meth:`~Cursor.execute` on it with the given *sql* and *parameters*.
      Return the new cursor object.

   .. method:: executemany(sql, parameters, /)

      Create a new :class:`Cursor` object and call
      :meth:`~Cursor.executemany` on it with the given *sql* and *parameters*.
      Return the new cursor object.

   .. method:: executescript(sql_script, /)

      Create a new :class:`Cursor` object and call
      :meth:`~Cursor.executescript` on it with the given *sql_script*.
      Return the new cursor object.

   .. method:: create_function(name, narg, func, \*, deterministic=False)

      Create or remove a user-defined SQL function.

      :param str name:
          The name of the SQL function.

      :param int narg:
          The number of arguments the SQL function can accept.
          If ``-1``, it may take any number of arguments.

      :param func:
          A callable that is called when the SQL function is invoked.
          The callable must return :ref:`a type natively supported by SQLite
          <sqlite3-types>`.
          Set to ``None`` to remove an existing SQL function.
      :type func: :term:`callback` | None

      :param bool deterministic:
          If ``True``, the created SQL function is marked as
          `deterministic <https://sqlite.org/deterministic.html>`_,
          which allows SQLite to perform additional optimizations.

      :raises NotSupportedError:
          If *deterministic* is used with SQLite versions older than 3.8.3.

      .. versionadded:: 3.8
         The *deterministic* parameter.

      Example:

      .. doctest::

         >>> import hashlib
         >>> def md5sum(t):
         ...     return hashlib.md5(t).hexdigest()
         >>> con = sqlite3.connect(":memory:")
         >>> con.create_function("md5", 1, md5sum)
         >>> for row in con.execute("SELECT md5(?)", (b"foo",)):
         ...     print(row)
         ('acbd18db4cc2f85cedef654fccc4a4d8',)


   .. method:: create_aggregate(name, /, n_arg, aggregate_class)

      Create or remove a user-defined SQL aggregate function.

      :param str name:
          The name of the SQL aggregate function.

      :param int n_arg:
          The number of arguments the SQL aggregate function can accept.
          If ``-1``, it may take any number of arguments.

      :param aggregate_class:
          A class must implement the following methods:

          * ``step()``: Add a row to the aggregate.
          * ``finalize()``: Return the final result of the aggregate as
            :ref:`a type natively supported by SQLite <sqlite3-types>`.

          The number of arguments that the ``step()`` method must accept
          is controlled by *n_arg*.

          Set to ``None`` to remove an existing SQL aggregate function.
      :type aggregate_class: :term:`class` | None

      Example:

      .. testcode::

         class MySum:
             def __init__(self):
                 self.count = 0

             def step(self, value):
                 self.count += value

             def finalize(self):
                 return self.count

         con = sqlite3.connect(":memory:")
         con.create_aggregate("mysum", 1, MySum)
         cur = con.execute("CREATE TABLE test(i)")
         cur.execute("INSERT INTO test(i) VALUES(1)")
         cur.execute("INSERT INTO test(i) VALUES(2)")
         cur.execute("SELECT mysum(i) FROM test")
         print(cur.fetchone()[0])

         con.close()

      .. testoutput::
         :hide:

         3


   .. method:: create_window_function(name, num_params, aggregate_class, /)

      Create or remove a user-defined aggregate window function.

      :param str name:
          The name of the SQL aggregate window function to create or remove.

      :param int num_params:
          The number of arguments the SQL aggregate window function can accept.
          If ``-1``, it may take any number of arguments.

      :param aggregate_class:
          A class that must implement the following methods:

          * ``step()``: Add a row to the current window.
          * ``value()``: Return the current value of the aggregate.
          * ``inverse()``: Remove a row from the current window.
          * ``finalize()``: Return the final result of the aggregate as
            :ref:`a type natively supported by SQLite <sqlite3-types>`.

          The number of arguments that the ``step()`` and ``value()`` methods
          must accept is controlled by *num_params*.

          Set to ``None`` to remove an existing SQL aggregate window function.

      :raises NotSupportedError:
          If used with a version of SQLite older than 3.25.0,
          which does not support aggregate window functions.

      :type aggregate_class: :term:`class` | None

      .. versionadded:: 3.11

      Example:

      .. testcode::

         # Example taken from https://www.sqlite.org/windowfunctions.html#udfwinfunc
         class WindowSumInt:
             def __init__(self):
                 self.count = 0

             def step(self, value):
                 """Add a row to the current window."""
                 self.count += value

             def value(self):
                 """Return the current value of the aggregate."""
                 return self.count

             def inverse(self, value):
                 """Remove a row from the current window."""
                 self.count -= value

             def finalize(self):
                 """Return the final value of the aggregate.

                 Any clean-up actions should be placed here.
                 """
                 return self.count


         con = sqlite3.connect(":memory:")
         cur = con.execute("CREATE TABLE test(x, y)")
         values = [
             ("a", 4),
             ("b", 5),
             ("c", 3),
             ("d", 8),
             ("e", 1),
         ]
         cur.executemany("INSERT INTO test VALUES(?, ?)", values)
         con.create_window_function("sumint", 1, WindowSumInt)
         cur.execute("""
             SELECT x, sumint(y) OVER (
                 ORDER BY x ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
             ) AS sum_y
             FROM test ORDER BY x
         """)
         print(cur.fetchall())

      .. testoutput::
         :hide:

         [('a', 9), ('b', 12), ('c', 16), ('d', 12), ('e', 9)]

   .. method:: create_collation(name, callable)

      Create a collation named *name* using the collating function *callable*.
      *callable* is passed two :class:`string <str>` arguments,
      and it should return an :class:`integer <int>`:

      * ``1`` if the first is ordered higher than the second
      * ``-1`` if the first is ordered lower than the second
      * ``0`` if they are ordered equal

      The following example shows a reverse sorting collation:

      .. testcode::

         def collate_reverse(string1, string2):
             if string1 == string2:
                 return 0
             elif string1 < string2:
                 return 1
             else:
                 return -1

         con = sqlite3.connect(":memory:")
         con.create_collation("reverse", collate_reverse)

         cur = con.execute("CREATE TABLE test(x)")
         cur.executemany("INSERT INTO test(x) VALUES(?)", [("a",), ("b",)])
         cur.execute("SELECT x FROM test ORDER BY x COLLATE reverse")
         for row in cur:
             print(row)
         con.close()

      .. testoutput::
         :hide:

         ('b',)
         ('a',)

      Remove a collation function by setting *callable* to ``None``.

      .. versionchanged:: 3.11
         The collation name can contain any Unicode character.  Earlier, only
         ASCII characters were allowed.


   .. method:: interrupt()

      Call this method from a different thread to abort any queries that might
      be executing on the connection.
      Aborted queries will raise an exception.


   .. method:: set_authorizer(authorizer_callback)

      Register callable *authorizer_callback* to be invoked for each attempt to
      access a column of a table in the database. The callback should return
      one of :const:`SQLITE_OK`, :const:`SQLITE_DENY`, or :const:`SQLITE_IGNORE`
      to signal how access to the column should be handled
      by the underlying SQLite library.

      The first argument to the callback signifies what kind of operation is to be
      authorized. The second and third argument will be arguments or ``None``
      depending on the first argument. The 4th argument is the name of the database
      ("main", "temp", etc.) if applicable. The 5th argument is the name of the
      inner-most trigger or view that is responsible for the access attempt or
      ``None`` if this access attempt is directly from input SQL code.

      Please consult the SQLite documentation about the possible values for the first
      argument and the meaning of the second and third argument depending on the first
      one. All necessary constants are available in the :mod:`!sqlite3` module.

      Passing ``None`` as *authorizer_callback* will disable the authorizer.

      .. versionchanged:: 3.11
         Added support for disabling the authorizer using ``None``.


   .. method:: set_progress_handler(progress_handler, n)

      Register callable *progress_handler* to be invoked for every *n*
      instructions of the SQLite virtual machine. This is useful if you want to
      get called from SQLite during long-running operations, for example to update
      a GUI.

      If you want to clear any previously installed progress handler, call the
      method with ``None`` for *progress_handler*.

      Returning a non-zero value from the handler function will terminate the
      currently executing query and cause it to raise a :exc:`DatabaseError`
      exception.


   .. method:: set_trace_callback(trace_callback)

      Register callable *trace_callback* to be invoked for each SQL statement
      that is actually executed by the SQLite backend.

      The only argument passed to the callback is the statement (as
      :class:`str`) that is being executed. The return value of the callback is
      ignored. Note that the backend does not only run statements passed to the
      :meth:`Cursor.execute` methods.  Other sources include the
      :ref:`transaction management <sqlite3-controlling-transactions>` of the
      :mod:`!sqlite3` module and the execution of triggers defined in the current
      database.

      Passing ``None`` as *trace_callback* will disable the trace callback.

      .. note::
         Exceptions raised in the trace callback are not propagated. As a
         development and debugging aid, use
         :meth:`~sqlite3.enable_callback_tracebacks` to enable printing
         tracebacks from exceptions raised in the trace callback.

      .. versionadded:: 3.3


   .. method:: enable_load_extension(enabled, /)

      Enable the SQLite engine to load SQLite extensions from shared libraries
      if *enabled* is ``True``;
      else, disallow loading SQLite extensions.
      SQLite extensions can define new functions,
      aggregates or whole new virtual table implementations.  One well-known
      extension is the fulltext-search extension distributed with SQLite.

      .. note::

         The :mod:`!sqlite3` module is not built with loadable extension support by
         default, because some platforms (notably macOS) have SQLite
         libraries which are compiled without this feature.
         To get loadable extension support,
         you must pass the :option:`--enable-loadable-sqlite-extensions` option
         to :program:`configure`.

      .. audit-event:: sqlite3.enable_load_extension connection,enabled sqlite3.Connection.enable_load_extension

      .. versionadded:: 3.2

      .. versionchanged:: 3.10
         Added the ``sqlite3.enable_load_extension`` auditing event.

      .. testsetup:: sqlite3.loadext

         import sqlite3
         con = sqlite3.connect(":memory:")

      .. testcode:: sqlite3.loadext
         :skipif: True  # not testable at the moment

         con.enable_load_extension(True)

         # Load the fulltext search extension
         con.execute("select load_extension('./fts3.so')")

         # alternatively you can load the extension using an API call:
         # con.load_extension("./fts3.so")

         # disable extension loading again
         con.enable_load_extension(False)

         # example from SQLite wiki
         con.execute("CREATE VIRTUAL TABLE recipe USING fts3(name, ingredients)")
         con.executescript("""
             INSERT INTO recipe (name, ingredients) VALUES('broccoli stew', 'broccoli peppers cheese tomatoes');
             INSERT INTO recipe (name, ingredients) VALUES('pumpkin stew', 'pumpkin onions garlic celery');
             INSERT INTO recipe (name, ingredients) VALUES('broccoli pie', 'broccoli cheese onions flour');
             INSERT INTO recipe (name, ingredients) VALUES('pumpkin pie', 'pumpkin sugar flour butter');
             """)
         for row in con.execute("SELECT rowid, name, ingredients FROM recipe WHERE name MATCH 'pie'"):
             print(row)

         con.close()

      .. testoutput:: sqlite3.loadext
         :hide:

         (2, 'broccoli pie', 'broccoli cheese onions flour')
         (3, 'pumpkin pie', 'pumpkin sugar flour butter')

   .. method:: load_extension(path, /)

      Load an SQLite extension from a shared library located at *path*.
      Enable extension loading with :meth:`enable_load_extension` before
      calling this method.

      .. audit-event:: sqlite3.load_extension connection,path sqlite3.Connection.load_extension

      .. versionadded:: 3.2

      .. versionchanged:: 3.10
         Added the ``sqlite3.load_extension`` auditing event.

   .. method:: iterdump

      Return an :term:`iterator` to dump the database as SQL source code.
      Useful when saving an in-memory database for later restoration.
      Similar to the ``.dump`` command in the :program:`sqlite3` shell.

      Example:

      .. testcode::

         # Convert file example.db to SQL dump file dump.sql
         con = sqlite3.connect('example.db')
         with open('dump.sql', 'w') as f:
             for line in con.iterdump():
                 f.write('%s\n' % line)
         con.close()


   .. method:: backup(target, \*, pages=-1, progress=None, name="main", sleep=0.250)

      Create a backup of an SQLite database.

      Works even if the database is being accessed by other clients
      or concurrently by the same connection.

      :param Connection target:
          The database connection to save the backup to.

      :param int pages:
          The number of pages to copy at a time.
          If equal to or less than ``0``,
          the entire database is copied in a single step.
          Defaults to ``-1``.

      :param progress:
          If set to a callable, it is invoked with three integer arguments for
          every backup iteration:
          the *status* of the last iteration,
          the *remaining* number of pages still to be copied,
          and the *total* number of pages.
          Defaults to ``None``.
      :type progress: :term:`callback` | None

      :param str name:
          The name of the database to back up.
          Either ``"main"`` (the default) for the main database,
          ``"temp"`` for the temporary database,
          or the name of a custom database as attached using the
          ``ATTACH DATABASE`` SQL statement.

      :param float sleep:
          The number of seconds to sleep between successive attempts
          to back up remaining pages.

      Example 1, copy an existing database into another:

      .. testcode::

         def progress(status, remaining, total):
             print(f'Copied {total-remaining} of {total} pages...')

         src = sqlite3.connect('example.db')
         dst = sqlite3.connect('backup.db')
         with dst:
             src.backup(dst, pages=1, progress=progress)
         dst.close()
         src.close()

      .. testoutput::
         :hide:

         Copied 0 of 0 pages...

      Example 2, copy an existing database into a transient copy:

      .. testcode::

         src = sqlite3.connect('example.db')
         dst = sqlite3.connect(':memory:')
         src.backup(dst)

      .. versionadded:: 3.7

   .. method:: getlimit(category, /)

      Get a connection runtime limit.

      :param int category:
         The `SQLite limit category`_ to be queried.

      :rtype: int

      :raises ProgrammingError:
         If *category* is not recognised by the underlying SQLite library.

      Example, query the maximum length of an SQL statement
      for :class:`Connection` ``con`` (the default is 1000000000):

      .. testsetup:: sqlite3.limits

         import sqlite3
         con = sqlite3.connect(":memory:")
         con.setlimit(sqlite3.SQLITE_LIMIT_SQL_LENGTH, 1_000_000_000)
         con.setlimit(sqlite3.SQLITE_LIMIT_ATTACHED, 10)

      .. doctest:: sqlite3.limits

         >>> con.getlimit(sqlite3.SQLITE_LIMIT_SQL_LENGTH)
         1000000000

      .. versionadded:: 3.11


   .. method:: setlimit(category, limit, /)

      Set a connection runtime limit.
      Attempts to increase a limit above its hard upper bound are silently
      truncated to the hard upper bound. Regardless of whether or not the limit
      was changed, the prior value of the limit is returned.

      :param int category:
         The `SQLite limit category`_ to be set.

      :param int limit:
         The value of the new limit.
         If negative, the current limit is unchanged.

      :rtype: int

      :raises ProgrammingError:
         If *category* is not recognised by the underlying SQLite library.

      Example, limit the number of attached databases to 1
      for :class:`Connection` ``con`` (the default limit is 10):

      .. doctest:: sqlite3.limits

         >>> con.setlimit(sqlite3.SQLITE_LIMIT_ATTACHED, 1)
         10
         >>> con.getlimit(sqlite3.SQLITE_LIMIT_ATTACHED)
         1

      .. versionadded:: 3.11

   .. _SQLite limit category: https://www.sqlite.org/c3ref/c_limit_attached.html


   .. method:: serialize(\*, name="main")

      Serialize a database into a :class:`bytes` object.  For an
      ordinary on-disk database file, the serialization is just a copy of the
      disk file.  For an in-memory database or a "temp" database, the
      serialization is the same sequence of bytes which would be written to
      disk if that database were backed up to disk.

      :param str name:
         The database name to be serialized.
         Defaults to ``"main"``.

      :rtype: bytes

      .. note::

         This method is only available if the underlying SQLite library has the
         serialize API.

      .. versionadded:: 3.11


   .. method:: deserialize(data, /, \*, name="main")

      Deserialize a :meth:`serialized <serialize>` database into a
      :class:`Connection`.
      This method causes the database connection to disconnect from database
      *name*, and reopen *name* as an in-memory database based on the
      serialization contained in *data*.

      :param bytes data:
         A serialized database.

      :param str name:
         The database name to deserialize into.
         Defaults to ``"main"``.

      :raises OperationalError:
         If the database connection is currently involved in a read
         transaction or a backup operation.

      :raises DatabaseError:
         If *data* does not contain a valid SQLite database.

      :raises OverflowError:
         If :func:`len(data) <len>` is larger than ``2**63 - 1``.

      .. note::

         This method is only available if the underlying SQLite library has the
         deserialize API.

      .. versionadded:: 3.11


.. _sqlite3-cursor-objects:

Cursor objects
^^^^^^^^^^^^^^

   A ``Cursor`` object represents a `database cursor`_
   which is used to execute SQL statements,
   and manage the context of a fetch operation.
   Cursors are created using :meth:`Connection.cursor`,
   or by using any of the :ref:`connection shortcut methods
   <sqlite3-connection-shortcuts>`.

   Cursor objects are :term:`iterators <iterator>`,
   meaning that if you :meth:`~Cursor.execute` a ``SELECT`` query,
   you can simply iterate over the cursor to fetch the resulting rows:

   .. testsetup:: sqlite3.cursor

      import sqlite3
      con = sqlite3.connect(":memory:", isolation_level=None)
      cur = con.execute("CREATE TABLE data(t)")
      cur.execute("INSERT INTO data VALUES(1)")

   .. testcode:: sqlite3.cursor

      for row in cur.execute("SELECT t FROM data"):
          print(row)

   .. testoutput:: sqlite3.cursor
      :hide:

      (1,)

   .. _database cursor: https://en.wikipedia.org/wiki/Cursor_(databases)

.. class:: Cursor

   A :class:`Cursor` instance has the following attributes and methods.

   .. index:: single: ? (question mark); in SQL statements
   .. index:: single: : (colon); in SQL statements

   .. method:: execute(sql, parameters=(), /)

      Execute SQL statement *sql*.
      Bind values to the statement using :ref:`placeholders
      <sqlite3-placeholders>` that map to the :term:`sequence` or :class:`dict`
      *parameters*.

      :meth:`execute` will only execute a single SQL statement. If you try to execute
      more than one statement with it, it will raise a :exc:`ProgrammingError`. Use
      :meth:`executescript` if you want to execute multiple SQL statements with one
      call.

      If :attr:`~Connection.isolation_level` is not ``None``,
      *sql* is an ``INSERT``, ``UPDATE``, ``DELETE``, or ``REPLACE`` statement,
      and there is no open transaction,
      a transaction is implicitly opened before executing *sql*.


   .. method:: executemany(sql, parameters, /)

      Execute :ref:`parameterized <sqlite3-placeholders>` SQL statement *sql*
      against all parameter sequences or mappings found in the sequence
      *parameters*.  It is also possible to use an
      :term:`iterator` yielding parameters instead of a sequence.
      Uses the same implicit transaction handling as :meth:`~Cursor.execute`.

      Example:

      .. testcode:: sqlite3.cursor

         rows = [
             ("row1",),
             ("row2",),
         ]
         # cur is an sqlite3.Cursor object
         cur.executemany("INSERT INTO data VALUES(?)", rows)

   .. method:: executescript(sql_script, /)

      Execute the SQL statements in *sql_script*.
      If there is a pending transaction,
      an implicit ``COMMIT`` statement is executed first.
      No other implicit transaction control is performed;
      any transaction control must be added to *sql_script*.

      *sql_script* must be a :class:`string <str>`.

      Example:

      .. testcode:: sqlite3.cursor

         # cur is an sqlite3.Cursor object
         cur.executescript("""
             BEGIN;
             CREATE TABLE person(firstname, lastname, age);
             CREATE TABLE book(title, author, published);
             CREATE TABLE publisher(name, address);
             COMMIT;
         """)


   .. method:: fetchone()

      Return the next row of a query result set as a :class:`tuple`.
      Return ``None`` if no more data is available.


   .. method:: fetchmany(size=cursor.arraysize)

      Return the next set of rows of a query result as a :class:`list`.
      Return an empty list if no more rows are available.

      The number of rows to fetch per call is specified by the *size* parameter.
      If *size* is not given, :attr:`arraysize` determines the number of rows
      to be fetched.
      If fewer than *size* rows are available,
      as many rows as are available are returned.

      Note there are performance considerations involved with the *size* parameter.
      For optimal performance, it is usually best to use the arraysize attribute.
      If the *size* parameter is used, then it is best for it to retain the same
      value from one :meth:`fetchmany` call to the next.

   .. method:: fetchall()

      Return all (remaining) rows of a query result as a :class:`list`.
      Return an empty list if no rows are available.
      Note that the :attr:`arraysize` attribute can affect the performance of
      this operation.

   .. method:: close()

      Close the cursor now (rather than whenever ``__del__`` is called).

      The cursor will be unusable from this point forward; a :exc:`ProgrammingError`
      exception will be raised if any operation is attempted with the cursor.

   .. method:: setinputsizes(sizes, /)

      Required by the DB-API. Does nothing in :mod:`!sqlite3`.

   .. method:: setoutputsize(size, column=None, /)

      Required by the DB-API. Does nothing in :mod:`!sqlite3`.

   .. attribute:: rowcount

      Read-only attribute that provides the number of modified rows for
      ``INSERT``, ``UPDATE``, ``DELETE``, and ``REPLACE`` statements;
      is ``-1`` for other statements,
      including :abbr:`CTE (Common Table Expression)` queries.
      It is only updated by the :meth:`execute` and :meth:`executemany` methods.

   .. attribute:: lastrowid

      Read-only attribute that provides the row id of the last inserted row. It
      is only updated after successful ``INSERT`` or ``REPLACE`` statements
      using the :meth:`execute` method.  For other statements, after
      :meth:`executemany` or :meth:`executescript`, or if the insertion failed,
      the value of ``lastrowid`` is left unchanged.  The initial value of
      ``lastrowid`` is ``None``.

      .. note::
         Inserts into ``WITHOUT ROWID`` tables are not recorded.

      .. versionchanged:: 3.6
         Added support for the ``REPLACE`` statement.

   .. attribute:: arraysize

      Read/write attribute that controls the number of rows returned by :meth:`fetchmany`.
      The default value is 1 which means a single row would be fetched per call.

   .. attribute:: description

      Read-only attribute that provides the column names of the last query. To
      remain compatible with the Python DB API, it returns a 7-tuple for each
      column where the last six items of each tuple are ``None``.

      It is set for ``SELECT`` statements without any matching rows as well.

   .. attribute:: connection

      Read-only attribute that provides the SQLite database :class:`Connection`
      belonging to the cursor.  A :class:`Cursor` object created by
      calling :meth:`con.cursor() <Connection.cursor>` will have a
      :attr:`connection` attribute that refers to *con*:

      .. doctest::

         >>> con = sqlite3.connect(":memory:")
         >>> cur = con.cursor()
         >>> cur.connection == con
         True

.. The sqlite3.Row example used to be a how-to. It has now been incorporated
   into the Row reference. We keep the anchor here in order not to break
   existing links.

.. _sqlite3-columns-by-name:
.. _sqlite3-row-objects:

Row objects
^^^^^^^^^^^

.. class:: Row

   A :class:`!Row` instance serves as a highly optimized
   :attr:`~Connection.row_factory` for :class:`Connection` objects.
   It supports iteration, equality testing, :func:`len`,
   and :term:`mapping` access by column name and index.

   Two row objects compare equal if have equal columns and equal members.

   .. method:: keys

      Return a :class:`list` of column names as :class:`strings <str>`.
      Immediately after a query,
      it is the first member of each tuple in :attr:`Cursor.description`.

   .. versionchanged:: 3.5
      Added support of slicing.

   Example:

   .. doctest::

      >>> con = sqlite3.connect(":memory:")
      >>> con.row_factory = sqlite3.Row
      >>> res = con.execute("SELECT 'Earth' AS name, 6378 AS radius")
      >>> row = res.fetchone()
      >>> row.keys()
      ['name', 'radius']
      >>> row[0], row["name"]  # Access by index and name.
      ('Earth', 'Earth')
      >>> row["RADIUS"]  # Column names are case-insensitive.
      6378


.. _sqlite3-blob-objects:

Blob objects
^^^^^^^^^^^^

.. versionadded:: 3.11

.. class:: Blob

   A :class:`Blob` instance is a :term:`file-like object`
   that can read and write data in an SQLite :abbr:`BLOB (Binary Large OBject)`.
   Call :func:`len(blob) <len>` to get the size (number of bytes) of the blob.
   Use indices and :term:`slices <slice>` for direct access to the blob data.

   Use the :class:`Blob` as a :term:`context manager` to ensure that the blob
   handle is closed after use.

   .. testcode::

      con = sqlite3.connect(":memory:")
      con.execute("CREATE TABLE test(blob_col blob)")
      con.execute("INSERT INTO test(blob_col) VALUES(zeroblob(13))")

      # Write to our blob, using two write operations:
      with con.blobopen("test", "blob_col", 1) as blob:
          blob.write(b"hello, ")
          blob.write(b"world.")
          # Modify the first and last bytes of our blob
          blob[0] = ord("H")
          blob[-1] = ord("!")

      # Read the contents of our blob
      with con.blobopen("test", "blob_col", 1) as blob:
          greeting = blob.read()

      print(greeting)  # outputs "b'Hello, world!'"

   .. testoutput::
      :hide:

      b'Hello, world!'

   .. method:: close()

      Close the blob.

      The blob will be unusable from this point onward.  An
      :class:`~sqlite3.Error` (or subclass) exception will be raised if any
      further operation is attempted with the blob.

   .. method:: read(length=-1, /)

      Read *length* bytes of data from the blob at the current offset position.
      If the end of the blob is reached, the data up to
      :abbr:`EOF (End of File)` will be returned.  When *length* is not
      specified, or is negative, :meth:`~Blob.read` will read until the end of
      the blob.

   .. method:: write(data, /)

      Write *data* to the blob at the current offset.  This function cannot
      change the blob length.  Writing beyond the end of the blob will raise
      :exc:`ValueError`.

   .. method:: tell()

      Return the current access position of the blob.

   .. method:: seek(offset, origin=os.SEEK_SET, /)

      Set the current access position of the blob to *offset*.  The *origin*
      argument defaults to :data:`os.SEEK_SET` (absolute blob positioning).
      Other values for *origin* are :data:`os.SEEK_CUR` (seek relative to the
      current position) and :data:`os.SEEK_END` (seek relative to the blob’s
      end).


PrepareProtocol objects
^^^^^^^^^^^^^^^^^^^^^^^

.. class:: PrepareProtocol

   The PrepareProtocol type's single purpose is to act as a :pep:`246` style
   adaption protocol for objects that can :ref:`adapt themselves
   <sqlite3-conform>` to :ref:`native SQLite types <sqlite3-types>`.


.. _sqlite3-exceptions:

Exceptions
^^^^^^^^^^

The exception hierarchy is defined by the DB-API 2.0 (:pep:`249`).

.. exception:: Warning

   This exception is not currently raised by the :mod:`!sqlite3` module,
   but may be raised by applications using :mod:`!sqlite3`,
   for example if a user-defined function truncates data while inserting.
   ``Warning`` is a subclass of :exc:`Exception`.

.. exception:: Error

   The base class of the other exceptions in this module.
   Use this to catch all errors with one single :keyword:`except` statement.
   ``Error`` is a subclass of :exc:`Exception`.

   If the exception originated from within the SQLite library,
   the following two attributes are added to the exception:

   .. attribute:: sqlite_errorcode

      The numeric error code from the
      `SQLite API <https://sqlite.org/rescode.html>`_

      .. versionadded:: 3.11

   .. attribute:: sqlite_errorname

      The symbolic name of the numeric error code
      from the `SQLite API <https://sqlite.org/rescode.html>`_

      .. versionadded:: 3.11

.. exception:: InterfaceError

   Exception raised for misuse of the low-level SQLite C API.
   In other words, if this exception is raised, it probably indicates a bug in the
   :mod:`!sqlite3` module.
   ``InterfaceError`` is a subclass of :exc:`Error`.

.. exception:: DatabaseError

   Exception raised for errors that are related to the database.
   This serves as the base exception for several types of database errors.
   It is only raised implicitly through the specialised subclasses.
   ``DatabaseError`` is a subclass of :exc:`Error`.

.. exception:: DataError

   Exception raised for errors caused by problems with the processed data,
   like numeric values out of range, and strings which are too long.
   ``DataError`` is a subclass of :exc:`DatabaseError`.

.. exception:: OperationalError

   Exception raised for errors that are related to the database's operation,
   and not necessarily under the control of the programmer.
   For example, the database path is not found,
   or a transaction could not be processed.
   ``OperationalError`` is a subclass of :exc:`DatabaseError`.

.. exception:: IntegrityError

   Exception raised when the relational integrity of the database is affected,
   e.g. a foreign key check fails.  It is a subclass of :exc:`DatabaseError`.

.. exception:: InternalError

   Exception raised when SQLite encounters an internal error.
   If this is raised, it may indicate that there is a problem with the runtime
   SQLite library.
   ``InternalError`` is a subclass of :exc:`DatabaseError`.

.. exception:: ProgrammingError

   Exception raised for :mod:`!sqlite3` API programming errors,
   for example supplying the wrong number of bindings to a query,
   or trying to operate on a closed :class:`Connection`.
   ``ProgrammingError`` is a subclass of :exc:`DatabaseError`.

.. exception:: NotSupportedError

   Exception raised in case a method or database API is not supported by the
   underlying SQLite library. For example, setting *deterministic* to
   ``True`` in :meth:`~Connection.create_function`, if the underlying SQLite library
   does not support deterministic functions.
   ``NotSupportedError`` is a subclass of :exc:`DatabaseError`.


.. _sqlite3-types:

SQLite and Python types
^^^^^^^^^^^^^^^^^^^^^^^

SQLite natively supports the following types: ``NULL``, ``INTEGER``,
``REAL``, ``TEXT``, ``BLOB``.

The following Python types can thus be sent to SQLite without any problem:

+-------------------------------+-------------+
| Python type                   | SQLite type |
+===============================+=============+
| ``None``                      | ``NULL``    |
+-------------------------------+-------------+
| :class:`int`                  | ``INTEGER`` |
+-------------------------------+-------------+
| :class:`float`                | ``REAL``    |
+-------------------------------+-------------+
| :class:`str`                  | ``TEXT``    |
+-------------------------------+-------------+
| :class:`bytes`                | ``BLOB``    |
+-------------------------------+-------------+


This is how SQLite types are converted to Python types by default:

+-------------+----------------------------------------------+
| SQLite type | Python type                                  |
+=============+==============================================+
| ``NULL``    | ``None``                                     |
+-------------+----------------------------------------------+
| ``INTEGER`` | :class:`int`                                 |
+-------------+----------------------------------------------+
| ``REAL``    | :class:`float`                               |
+-------------+----------------------------------------------+
| ``TEXT``    | depends on :attr:`~Connection.text_factory`, |
|             | :class:`str` by default                      |
+-------------+----------------------------------------------+
| ``BLOB``    | :class:`bytes`                               |
+-------------+----------------------------------------------+

The type system of the :mod:`!sqlite3` module is extensible in two ways: you can
store additional Python types in an SQLite database via
:ref:`object adapters <sqlite3-adapters>`,
and you can let the :mod:`!sqlite3` module convert SQLite types to
Python types via :ref:`converters <sqlite3-converters>`.


.. _sqlite3-default-converters:

Default adapters and converters (deprecated)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. note::

   The default adapters and converters are deprecated as of Python 3.12.
   Instead, use the :ref:`sqlite3-adapter-converter-recipes`
   and tailor them to your needs.

The deprecated default adapters and converters consist of:

* An adapter for :class:`datetime.date` objects to :class:`strings <str>` in
  `ISO 8601`_ format.
* An adapter for :class:`datetime.datetime` objects to strings in
  ISO 8601 format.
* A converter for :ref:`declared <sqlite3-converters>` "date" types to
  :class:`datetime.date` objects.
* A converter for declared "timestamp" types to
  :class:`datetime.datetime` objects.
  Fractional parts will be truncated to 6 digits (microsecond precision).

.. note::

   The default "timestamp" converter ignores UTC offsets in the database and
   always returns a naive :class:`datetime.datetime` object. To preserve UTC
   offsets in timestamps, either leave converters disabled, or register an
   offset-aware converter with :func:`register_converter`.

.. deprecated:: 3.12

.. _ISO 8601: https://en.wikipedia.org/wiki/ISO_8601


.. _sqlite3-cli:

Command-line interface
^^^^^^^^^^^^^^^^^^^^^^

The :mod:`!sqlite3` module can be invoked as a script
in order to provide a simple SQLite shell.
Type ``.quit`` or CTRL-D to exit the shell.

.. program:: python -m sqlite3 [-h] [-v] [filename] [sql]

.. option:: -h, --help

   Print CLI help.

.. option:: -v, --version

   Print underlying SQLite library version.

.. versionadded:: 3.12


.. _sqlite3-howtos:

How-to guides
-------------

.. _sqlite3-placeholders:

How to use placeholders to bind values in SQL queries
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SQL operations usually need to use values from Python variables. However,
beware of using Python's string operations to assemble queries, as they
are vulnerable to `SQL injection attacks`_ (see the `xkcd webcomic
<https://xkcd.com/327/>`_ for a humorous example of what can go wrong)::

   # Never do this -- insecure!
   symbol = 'RHAT'
   cur.execute("SELECT * FROM stocks WHERE symbol = '%s'" % symbol)

Instead, use the DB-API's parameter substitution. To insert a variable into a
query string, use a placeholder in the string, and substitute the actual values
into the query by providing them as a :class:`tuple` of values to the second
argument of the cursor's :meth:`~Cursor.execute` method. An SQL statement may
use one of two kinds of placeholders: question marks (qmark style) or named
placeholders (named style). For the qmark style, ``parameters`` must be a
:term:`sequence <sequence>`. For the named style, it can be either a
:term:`sequence <sequence>` or :class:`dict` instance. The length of the
:term:`sequence <sequence>` must match the number of placeholders, or a
:exc:`ProgrammingError` is raised. If a :class:`dict` is given, it must contain
keys for all named parameters. Any extra items are ignored. Here's an example of
both styles:

.. testcode::

   con = sqlite3.connect(":memory:")
   cur = con.execute("CREATE TABLE lang(name, first_appeared)")

   # This is the qmark style:
   cur.execute("INSERT INTO lang VALUES(?, ?)", ("C", 1972))

   # The qmark style used with executemany():
   lang_list = [
       ("Fortran", 1957),
       ("Python", 1991),
       ("Go", 2009),
   ]
   cur.executemany("INSERT INTO lang VALUES(?, ?)", lang_list)

   # And this is the named style:
   cur.execute("SELECT * FROM lang WHERE first_appeared = :year", {"year": 1972})
   print(cur.fetchall())

.. testoutput::
   :hide:

   [('C', 1972)]


.. _sqlite3-adapters:

How to adapt custom Python types to SQLite values
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SQLite supports only a limited set of data types natively.
To store custom Python types in SQLite databases, *adapt* them to one of the
:ref:`Python types SQLite natively understands <sqlite3-types>`.

There are two ways to adapt Python objects to SQLite types:
letting your object adapt itself, or using an *adapter callable*.
The latter will take precedence above the former.
For a library that exports a custom type,
it may make sense to enable that type to adapt itself.
As an application developer, it may make more sense to take direct control by
registering custom adapter functions.


.. _sqlite3-conform:

How to write adaptable objects
""""""""""""""""""""""""""""""

Suppose we have a :class:`!Point` class that represents a pair of coordinates,
``x`` and ``y``, in a Cartesian coordinate system.
The coordinate pair will be stored as a text string in the database,
using a semicolon to separate the coordinates.
This can be implemented by adding a ``__conform__(self, protocol)``
method which returns the adapted value.
The object passed to *protocol* will be of type :class:`PrepareProtocol`.

.. testcode::

   class Point:
       def __init__(self, x, y):
           self.x, self.y = x, y

       def __conform__(self, protocol):
           if protocol is sqlite3.PrepareProtocol:
               return f"{self.x};{self.y}"

   con = sqlite3.connect(":memory:")
   cur = con.cursor()

   cur.execute("SELECT ?", (Point(4.0, -3.2),))
   print(cur.fetchone()[0])

.. testoutput::
   :hide:

   4.0;-3.2


How to register adapter callables
"""""""""""""""""""""""""""""""""

The other possibility is to create a function that converts the Python object
to an SQLite-compatible type.
This function can then be registered using :func:`register_adapter`.

.. testcode::

   class Point:
       def __init__(self, x, y):
           self.x, self.y = x, y

   def adapt_point(point):
       return f"{point.x};{point.y}"

   sqlite3.register_adapter(Point, adapt_point)

   con = sqlite3.connect(":memory:")
   cur = con.cursor()

   cur.execute("SELECT ?", (Point(1.0, 2.5),))
   print(cur.fetchone()[0])

.. testoutput::
   :hide:

   1.0;2.5


.. _sqlite3-converters:

How to convert SQLite values to custom Python types
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Writing an adapter lets you convert *from* custom Python types *to* SQLite
values.
To be able to convert *from* SQLite values *to* custom Python types,
we use *converters*.

Let's go back to the :class:`!Point` class. We stored the x and y coordinates
separated via semicolons as strings in SQLite.

First, we'll define a converter function that accepts the string as a parameter
and constructs a :class:`!Point` object from it.

.. note::

   Converter functions are **always** passed a :class:`bytes` object,
   no matter the underlying SQLite data type.

.. testcode::

   def convert_point(s):
       x, y = map(float, s.split(b";"))
       return Point(x, y)

We now need to tell :mod:`!sqlite3` when it should convert a given SQLite value.
This is done when connecting to a database, using the *detect_types* parameter
of :func:`connect`. There are three options:

* Implicit: set *detect_types* to :const:`PARSE_DECLTYPES`
* Explicit: set *detect_types* to :const:`PARSE_COLNAMES`
* Both: set *detect_types* to
  ``sqlite3.PARSE_DECLTYPES | sqlite3.PARSE_COLNAMES``.
  Column names take precedence over declared types.

The following example illustrates the implicit and explicit approaches:

.. testcode::

   class Point:
       def __init__(self, x, y):
           self.x, self.y = x, y

       def __repr__(self):
           return f"Point({self.x}, {self.y})"

   def adapt_point(point):
       return f"{point.x};{point.y}".encode("utf-8")

   def convert_point(s):
       x, y = list(map(float, s.split(b";")))
       return Point(x, y)

   # Register the adapter and converter
   sqlite3.register_adapter(Point, adapt_point)
   sqlite3.register_converter("point", convert_point)

   # 1) Parse using declared types
   p = Point(4.0, -3.2)
   con = sqlite3.connect(":memory:", detect_types=sqlite3.PARSE_DECLTYPES)
   cur = con.execute("CREATE TABLE test(p point)")

   cur.execute("INSERT INTO test(p) VALUES(?)", (p,))
   cur.execute("SELECT p FROM test")
   print("with declared types:", cur.fetchone()[0])
   cur.close()
   con.close()

   # 2) Parse using column names
   con = sqlite3.connect(":memory:", detect_types=sqlite3.PARSE_COLNAMES)
   cur = con.execute("CREATE TABLE test(p)")

   cur.execute("INSERT INTO test(p) VALUES(?)", (p,))
   cur.execute('SELECT p AS "p [point]" FROM test')
   print("with column names:", cur.fetchone()[0])

.. testoutput::
   :hide:

   with declared types: Point(4.0, -3.2)
   with column names: Point(4.0, -3.2)


.. _sqlite3-adapter-converter-recipes:

Adapter and converter recipes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This section shows recipes for common adapters and converters.

.. testcode::

   import datetime
   import sqlite3

   def adapt_date_iso(val):
       """Adapt datetime.date to ISO 8601 date."""
       return val.isoformat()

   def adapt_datetime_iso(val):
       """Adapt datetime.datetime to timezone-naive ISO 8601 date."""
       return val.isoformat()

   def adapt_datetime_epoch(val):
       """Adapt datetime.datetime to Unix timestamp."""
       return int(val.timestamp())

   sqlite3.register_adapter(datetime.date, adapt_date_iso)
   sqlite3.register_adapter(datetime.datetime, adapt_datetime_iso)
   sqlite3.register_adapter(datetime.datetime, adapt_datetime_epoch)

   def convert_date(val):
       """Convert ISO 8601 date to datetime.date object."""
       return datetime.date.fromisoformat(val)

   def convert_datetime(val):
       """Convert ISO 8601 datetime to datetime.datetime object."""
       return datetime.datetime.fromisoformat(val)

   def convert_timestamp(val):
       """Convert Unix epoch timestamp to datetime.datetime object."""
       return datetime.datetime.fromtimestamp(val)

   sqlite3.register_converter("date", convert_date)
   sqlite3.register_converter("datetime", convert_datetime)
   sqlite3.register_converter("timestamp", convert_timestamp)


.. _sqlite3-connection-shortcuts:

How to use connection shortcut methods
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Using the :meth:`~Connection.execute`,
:meth:`~Connection.executemany`, and :meth:`~Connection.executescript`
methods of the :class:`Connection` class, your code can
be written more concisely because you don't have to create the (often
superfluous) :class:`Cursor` objects explicitly. Instead, the :class:`Cursor`
objects are created implicitly and these shortcut methods return the cursor
objects. This way, you can execute a ``SELECT`` statement and iterate over it
directly using only a single call on the :class:`Connection` object.

.. testcode::

   # Create and fill the table.
   con = sqlite3.connect(":memory:")
   con.execute("CREATE TABLE lang(name, first_appeared)")
   data = [
       ("C++", 1985),
       ("Objective-C", 1984),
   ]
   con.executemany("INSERT INTO lang(name, first_appeared) VALUES(?, ?)", data)

   # Print the table contents
   for row in con.execute("SELECT name, first_appeared FROM lang"):
       print(row)

   print("I just deleted", con.execute("DELETE FROM lang").rowcount, "rows")

   # close() is not a shortcut method and it's not called automatically;
   # the connection object should be closed manually
   con.close()

.. testoutput::
   :hide:

   ('C++', 1985)
   ('Objective-C', 1984)
   I just deleted 2 rows


.. _sqlite3-connection-context-manager:

How to use the connection context manager
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A :class:`Connection` object can be used as a context manager that
automatically commits or rolls back open transactions when leaving the body of
the context manager.
If the body of the :keyword:`with` statement finishes without exceptions,
the transaction is committed.
If this commit fails,
or if the body of the ``with`` statement raises an uncaught exception,
the transaction is rolled back.

If there is no open transaction upon leaving the body of the ``with`` statement,
the context manager is a no-op.

.. note::

   The context manager neither implicitly opens a new transaction
   nor closes the connection.

.. testcode::

   con = sqlite3.connect(":memory:")
   con.execute("CREATE TABLE lang(id INTEGER PRIMARY KEY, name VARCHAR UNIQUE)")

   # Successful, con.commit() is called automatically afterwards
   with con:
       con.execute("INSERT INTO lang(name) VALUES(?)", ("Python",))

   # con.rollback() is called after the with block finishes with an exception,
   # the exception is still raised and must be caught
   try:
       with con:
           con.execute("INSERT INTO lang(name) VALUES(?)", ("Python",))
   except sqlite3.IntegrityError:
       print("couldn't add Python twice")

   # Connection object used as context manager only commits or rollbacks transactions,
   # so the connection object should be closed manually
   con.close()

.. testoutput::
   :hide:

   couldn't add Python twice


.. _sqlite3-uri-tricks:

How to work with SQLite URIs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Some useful URI tricks include:

* Open a database in read-only mode:

.. doctest::

   >>> con = sqlite3.connect("file:tutorial.db?mode=ro", uri=True)
   >>> con.execute("CREATE TABLE readonly(data)")
   Traceback (most recent call last):
   OperationalError: attempt to write a readonly database

* Do not implicitly create a new database file if it does not already exist;
  will raise :exc:`~sqlite3.OperationalError` if unable to create a new file:

.. doctest::

   >>> con = sqlite3.connect("file:nosuchdb.db?mode=rw", uri=True)
   Traceback (most recent call last):
   OperationalError: unable to open database file


* Create a shared named in-memory database:

.. testcode::

   db = "file:mem1?mode=memory&cache=shared"
   con1 = sqlite3.connect(db, uri=True)
   con2 = sqlite3.connect(db, uri=True)
   with con1:
       con1.execute("CREATE TABLE shared(data)")
       con1.execute("INSERT INTO shared VALUES(28)")
   res = con2.execute("SELECT data FROM shared")
   assert res.fetchone() == (28,)


More information about this feature, including a list of parameters,
can be found in the `SQLite URI documentation`_.

.. _SQLite URI documentation: https://www.sqlite.org/uri.html


.. _sqlite3-explanation:

Explanation
-----------

.. _sqlite3-controlling-transactions:

Transaction control
^^^^^^^^^^^^^^^^^^^

The :mod:`!sqlite3` module does not adhere to the transaction handling recommended
by :pep:`249`.

If the connection attribute :attr:`~Connection.isolation_level`
is not ``None``,
new transactions are implicitly opened before
:meth:`~Cursor.execute` and :meth:`~Cursor.executemany` executes
``INSERT``, ``UPDATE``, ``DELETE``, or ``REPLACE`` statements.
Use the :meth:`~Connection.commit` and :meth:`~Connection.rollback` methods
to respectively commit and roll back pending transactions.
You can choose the underlying `SQLite transaction behaviour`_ —
that is, whether and what type of ``BEGIN`` statements :mod:`!sqlite3`
implicitly executes –
via the :attr:`~Connection.isolation_level` attribute.

If :attr:`~Connection.isolation_level` is set to ``None``,
no transactions are implicitly opened at all.
This leaves the underlying SQLite library in `autocommit mode`_,
but also allows the user to perform their own transaction handling
using explicit SQL statements.
The underlying SQLite library autocommit mode can be queried using the
:attr:`~Connection.in_transaction` attribute.

The :meth:`~Cursor.executescript` method implicitly commits
any pending transaction before execution of the given SQL script,
regardless of the value of :attr:`~Connection.isolation_level`.

.. versionchanged:: 3.6
   :mod:`!sqlite3` used to implicitly commit an open transaction before DDL
   statements.  This is no longer the case.

.. _autocommit mode:
   https://www.sqlite.org/lang_transaction.html#implicit_versus_explicit_transactions

.. _SQLite transaction behaviour:
   https://www.sqlite.org/lang_transaction.html#deferred_immediate_and_exclusive_transactions
