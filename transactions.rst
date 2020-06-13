.. _db_session:

Transactions and db_session
===========================

A database transaction is a logical unit of work, which can consist of one or several queries. Transactions are atomic, which means that when a transaction makes changes to the database, either all the changes succeed when the transaction is committed, or all the changes are undone when the transaction is rolled back.

Pony provides automatic transaction management using the database session.


Working with db_session
-----------------------

The code which interacts with the database has to be placed within a database session. The session sets the borders of a conversation with the database. Each application thread which works with the database establishes a separate database session and uses a separate instance of an `Identity Map <https://martinfowler.com/eaaCatalog/identityMap.html>`_. This Identity Map works as a cache, helping to avoid a database query when you access an object by its primary or unique key and it is already stored in the Identity Map.
In order to work with the database using the database session you can use the :py:func:`@db_session` decorator or :py:func:`db_session` context manager. When the session ends it does the following actions:

* Commits transaction if data was changed and no exceptions occurred otherwise it rolls back transaction.
* Returns the database connection to the connection pool.
* Clears the Identity Map cache.

If you forget to specify the :py:func:`db_session` where necessary, Pony will raise the exception ``TransactionError: db_session is required when working with the database``.

Example of using the :py:func:`@db_session` decorator:

.. code-block:: python

    @db_session
    def check_user(username):
        return User.exists(username=username)

Example of using the :py:func:`db_session` context manager:

.. only:: comment
    Where does ``username`` come from?
    Does the change work?

.. code-block:: python

    def process_request(username):
        ...
        with db_session:
            u = User.get(username=username)
            ...

.. note::
   When you work with Python’s interactive shell you don’t need to worry about the database session, because it is maintained by Pony automatically.

If you'll try to access instance's attributes which were not loaded from the database outside of the :py:func:`db_session` scope, you'll get the ``DatabaseSessionIsOver`` exception. For example:

.. code-block:: text

    DatabaseSessionIsOver: Cannot load attribute Customer[3].name: the database session is over

This happens because by this moment the connection to the database is already returned to the connection pool, transaction is closed and we cannot send any queries to the database.

When Pony reads objects from the database it puts those objects to the Identity Map. Later, when you update an object’s attributes, create or delete an object, the changes will be accumulated in the Identity Map first. The changes will be saved in the database on transaction commit or before calling the following methods: ``select()``, ``get()``, ``exists()``, ``execute()``.


db_session and the transaction scope
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Usually you will have a single transaction within the :py:func:`db_session`. There is no explicit command for starting a transaction. A transaction begins with the first SQL query sent to the database. Before sending the first query, Pony gets a database connection from the connection pool. Any following SQL queries will be executed in the context of the same transaction.

.. note::
   Python driver for SQLite doesn’t start a transaction on a SELECT statement. It only begins a transaction on a statement which can modify the database: INSERT, UPDATE, DELETE. Other drivers start a transaction on any SQL statement, including SELECT.


A transaction ends when it is committed or rolled back using ``commit()`` or ``rollback()`` calls or implicitly by leaving the :py:func:`db_session` scope.

.. code-block:: python

    @db_session
    def func():
        # a new transaction is started
        p = Product[123]
        p.price += 10
        # commit() will be done automatically
        # database session cache will be cleared automatically
        # database connection will be returned to the pool


Several transactions within the same db_session
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you need to have more than one transaction within the same database session you can call ``commit()`` or ``rollback()`` at any time during the session, and then the next query will start a new transaction. The Identity Map keeps data after the manual ``commit()``, but if you call ``rollback()`` the cache will be cleared.

.. code-block:: python

    @db_session
    def func1():
        p1 = Product[123]
        p1.price += 10
        commit()          # the first transaction is committed
        p2 = Product[456] # a new transaction is started
        p2.price -= 10


Nested db_session
~~~~~~~~~~~~~~~~~

If you enter the :py:func:`db_session` scope recursively, for example by calling a function which is decorated with the :py:func:`db_session` decorator from another function which is decorated with :py:func:`db_session`, Pony will not create a new session, but will share the same session for both functions. The database session ends on leaving the scope of the outermost :py:func:`db_session` decorator or context manager.

What if inner :py:func:`db_session` has different settings? For example, the outer one is a default :py:func:`db_session` and the inner one is defined as ``db_session(optimistic=False)``?

Currently Pony checks inner :py:func:`db_session` options, and does one of the following:

1. If inner :py:func:`db_session` uses options incompatible with the outer :py:func:`db_session` (``ddl=True`` or ``serializable=True``), Pony throws an exception.
2. For ``sql_debug`` option Pony uses new sql_debug option value inside the inner :py:func:`db_session` and restores it when returning to the outer :py:func:`db_session`.
3. Other options (``strict``, ``optimistic``, ``immediate`` and ``retry``) are ignored for the inner :py:func:`db_session`.

If :py:func:`rollback` is called inside inner :py:func:`db_session`, it will be applied to the outer :py:func:`db_session`.

Some databases support nested transactions, but at the moment Pony doesn't support this.


db_session cache
~~~~~~~~~~~~~~~~

Pony caches data at several stages for increasing performance. It caches:

* The results of a generator expression translation. If the same generator expression query is used several times within the program, it will be translated to SQL only once. This cache is global for entire program, not only for a single database session.
* Objects which were created or loaded from the database. Pony keeps these objects in the Identity Map. This cache is cleared on leaving the :py:func:`db_session` scope or on transaction rollback.
* Query results. Pony returns the query result from the cache if the same query is called with the same parameters once again. This cache is cleared once any of entity instances is changed. This cache is cleared on leaving the :py:func:`db_session` scope or on transaction rollback.


Using db_session with generator functions or coroutines
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. only:: comment
    Should use official pydoc explanation, or at least link to it

The :py:func:`@db_session` decorator can be used with generator functions or coroutines too. A generator is a function that contains the ``yield`` keyword inside it. A coroutine is a function which is defined using ``async def`` or decorated with ``@asyncio.coroutine``.

If inside such a generator function or coroutine you try to use the ``db_session`` context manager, it will not work properly, because in Python, context managers cannot intercept generator suspension. Instead, you need to wrap you generator function or coroutine with the ``@db_session`` decorator.

In other words, don't do this:

.. code-block:: python

    def my_generator(x):
        with db_session: # transaction will never end
            obj = MyEntity.get(id=x)
            yield obj

Do this instead:

.. only:: comment
    Any reason for inconsistent formatting

.. code-block:: python

    @db_session
    def my_generator( x ):
        obj = MyEntity.get(id=x)
        yield obj

With regular functions, the :py:func:`@db_session` decorator works as a scope. When your program leaves the :py:func:`db_session` scope, Pony finishes the transaction by performing commit (or rollback) and clears the db_session cache.

In case of a generator, the program can reenter the generator code for several times. In this case, when your program leaves the generator code, the db_session is not over, but suspended and Pony doesn't clear the cache. In the same time, we don't know if the program will come back to this generator code again. That is why you have to explicitly commit or rollback current transaction before the program leaves the generator on ``yield``. On regular functions Pony calls ``commit()`` or ``rollback()`` automatically on leaving the :py:func:`db_session` scope.

In essence, here is the difference when using :py:func:`db_session` with generator functions:

1. You have to call ``commit()`` or ``rollback()`` before the ``yield`` expression explicitly.
2. Pony doesn't clear the transaction cache, so you can continue using loaded objects when coming back to the same generator.
3. With a generator function, the :py:func:`db_session` can be used only as a decorator, not a context manager. This is because in Python the context manager cannot understand that it was left on ``yield``.
4. The :py:func:`db_session` parameters, such as ``retry``, ``serializable`` cannot be used with generator functions. The only parameter that can be used in this case is ``immediate``.


Parameters of db_session
~~~~~~~~~~~~~~~~~~~~~~~~

As mentioned above, :py:func:`db_session` can be used as a decorator or a context manager. It can receive parameters which are described in the :ref:`API Reference <db_session>`.



Working with multiple databases
-------------------------------

Pony can work with several databases simultaneously. In the example below we use PostgreSQL for storing user information and MySQL for storing information about addresses:

.. code-block:: python

    db1 = Database()

    class User(db1.Entity):
        ...

    db1.bind('postgres', ...)


    db2 = Database()

    class Address(db2.Entity):
        ...

    db2.bind('mysql', ...)

    @db_session
    def do_something(user_id, address_id):
        u = User[user_id]
        a = Address[address_id]
        ...

On exiting from the ``do_something()`` function Pony will perform ``commit()`` or ``rollback()`` to both databases. If you need to commit to one database before exiting from the function you can use ``db1.commit()`` or ``db2.commit()`` methods.


Functions for working with transactions
---------------------------------------

There are three upper level functions that you can use for working with transactions:

* :py:func:`commit`
* :py:func:`rollback`
* :py:func:`flush`

Also there are three corresponding functions of the :py:class:`Database` object:

* :py:meth:`Database.commit`
* :py:meth:`Database.rollback`
* :py:meth:`Database.flush`

If you work with one database, there is no difference between using the upper level function or the :py:class:`Database` object methods.

.. _optimistic_control:

Optimistic concurrency control
------------------------------

By default Pony uses the optimistic concurrency control concept for increasing performance. With this concept, Pony doesn’t acquire locks on database rows. Instead it verifies that no other transaction has modified the data it has read or is trying to modify. If the check reveals conflicting modifications, the committing transaction gets the exception ``OptimisticCheckError, 'Object XYZ was updated outside of current transaction'`` and rolls back.

What should we do with this situation? First of all, this behavior is normal for databases which implement the `MVCC <http://en.wikipedia.org/wiki/Multiversion_concurrency_control>`_ pattern (e.g. Postgres, Oracle). For example, in Postgres, you will get the following error when a concurrent transaction changed the same data:

.. code-block:: text

    ERROR:  could not serialize access due to concurrent update

The current transaction rolls back, but it can be restarted. In order to restart the transaction automatically, you can use the ``retry`` parameter of the :py:func:`db_session` decorator (see more details about it later in this chapter).

Pony performs this optimistic check by tracking access to attributes of each object. If code reads or modifies an object’s attribute, Pony will then check if this attribute value is unchanged in the database before committing. This approach guarantees that there will be no lost updates. Take for example the situation where, during the current transaction, another transaction changes the same object after the current transaction reads it, but before the transaction i committed. Without optimistic checking the current transaction overwrites the data without knowing there were changes.

During the optimistic check Pony verifies only those attributes which were read or written by the user. Also when Pony updates an object, it updates only those attributes which were changed by the user. This way it is possible to have two concurrent transactions which change different attributes of the same object and both of them succeed.

Generally the optimistic concurrency control increases the performance because transactions can complete without the expense of managing locks or without having transactions wait for other transactions’ lock to clear. This approach shows very good results when conflicts are rare and our application reads data more often then writes.

However, if contention for writing data is frequent, the cost of repeatedly restarting transactions hurts performance. In this case the pessimistic locking can be more appropriate.

If you need to turn the optimistic concurrency control for an attribute off, you can use the :ref:`optimistic option <optimistic_option>` or :ref:`volatile option <volatile_option>`.


Pessimistic locking
-------------------

Sometimes we need to lock an object in the database in order to prevent other transactions from modifying the same record. Within the database such a lock should be done using the SELECT FOR UPDATE query. In order to generate such a lock using Pony you should call the :py:meth:`Query.for_update` method:

.. code-block:: python

    select(p for p in Product if p.price > 100).for_update()

The query above selects all instances of Product with the price greater than 100 and locks the corresponding rows in the database. The lock will be released upon commit or rollback of current transaction.

If you need to lock a single object, you can use the ``get_for_update`` method of an entity:

.. code-block:: python

    Product.get_for_update(id=123)

When trying to lock an object using :py:meth:`~Query.for_update` that is already locked by another transaction, your request will need to wait until the row-level lock is released. To prevent the operation from waiting for other transactions to commit, use the ``nowait=True`` option:

.. code-block:: python

    select(p for p in Product if p.price > 100).for_update(nowait=True)
    # or
    Product.get_for_update(id=123, nowait=True)

.. only:: comment
    What error message is raised?
    Can it be caught in a try/except block?

In this case, if a selected row(s) cannot be locked immediately, the request reports an error, rather than waiting.

The main disadvantage of pessimistic locking is performance degradation from using database locks and limiting concurrency.


How Pony avoids lost updates
----------------------------

Lower isolation levels increase the ability of many users to access data at the same time, but it also can lead to database anomalies such as lost updates. 

Let’s consider an example. Say we have two accounts. We need to provide a function which can transfer money from one account to another. During the transfer we check if the account has enough funds.

Let’s say we are using Django ORM for this task. Below is one of the possible ways of implementing such a function:

.. code-block:: python

    def transfer_money(from_account_id, to_account_id, amount):
        from_account = Account.objects.get(pk=from_account_id)
        to_account = Account.objects.get(pk=to_account_id)
        if amount > from_account.amount:    # validation
            raise ValueError("Not enough funds")
        from_account.amount -= amount
        from_account.save()
        to_account.amount += amount
        to_account.save()


By default in Django, each ``save()`` is performed in a separate transaction. If after the first ``save()`` there is a failure, the transaction won't be completed, and the money debited from the ``from_account`` will disappear. Even if no failure is encountered, if another transaction gets the ``to_account`` statement in between the two ``save()`` operations, the result will be wrong. To avoid such problems, both operations should be combined into one transaction. We can do that by decorating the function with the ``@transaction.atomic`` decorator.

.. code-block:: python

    @transaction.atomic
    def transfer_money(from_account_id, to_account_id, amount):
        from_account = Account.objects.get(pk=from_account_id)
        to_account = Account.objects.get(pk=to_account_id)
        if amount > from_account.amount:    # validation
            raise ValueError("Not enough funds")
        from_account.amount -= amount
        from_account.save()
        to_account.amount += amount
        to_account.save()


But even in this case we can encounter a problem. If two bank branches try to transfer the full amount to different accounts at the same time, both operations will be performed. Each function will pass the validation and finally one transaction will override the results of another one. This anomaly is called a "lost update".

There are three ways to prevent such an anomaly:

* Use the SERIALIZABLE isolation level
* Use SELECT FOR UPDATE instead SELECT
* Use optimistic checks

If you use the SERIALIZABLE isolation level, the database will not allow the second transaction to be committed, and will throw an exception upon calling ``commit()``. The disadvantage of such an approach is that this level requires more system resources.

If you use SELECT FOR UPDATE then the transaction which hits the database first will lock the row and all other transactions modifying the same row will be forced to wait.

The optimistic check doesn’t require more system resources and doesn’t lock the database rows. It eliminates the lost update anomaly by ensuring that the data wasn’t changed between the moment when we read it from the database and the commit operation.

The only way to avoid the lost update anomaly in Django is by using SELECT FOR UPDATE, which you should use explicitly. If you forget to do that, or if you don’t realize that your business logic could potentially encounter this lost update problem, your data can be lost.

Pony allows using all three approaches, having the third one, optimistic checks, used by default. This way Pony avoids the lost update anomaly completely. Also using the optimistic checks allows the highest concurrency because it doesn’t lock the database and doesn’t require extra resources.

.. only:: comment
    In database terminology, what is the name for the general class of
    what is being discussed here? "atomicity techniques", "contention
    detection", "concurrency guards"?

A similar function to transfer money can be written using Pony. This is what it would look like using all three available techniques.

The SERIALIZABLE approach:

.. code-block:: python

    @db_session(serializable=True)
    def transfer_money(from_account_id, to_account_id, amount):
        from_account = Account[from_account_id]
        to_account   = Account[to_account_id  ]
        if amount > from_account.amount:
            raise ValueError("Not enough funds")
        from_account.amount -= amount
        to_account.amount   += amount


The SELECT FOR UPDATE approach:

.. code-block:: python

    @db_session
    def transfer_money(from_account_id, to_account_id, amount):
        from_account = Account.get_for_update(id=from_account_id)
        to_account   = Account.get_for_update(id=to_account_id  )
        if amount > from_account.amount:
            raise ValueError("Not enough funds")
        from_account.amount -= amount
        to_account.amount   += amount

The optimistic check approach:

.. code-block:: python

    @db_session
    def transfer_money(account_id1, account_id2, amount):
        account1 = Account[account_id1]
        account2 = Account[account_id2]
        if amount > account1.amount:
            raise ValueError("Not enough funds")
        account1.amount -= amount
        account2.amount += amount


The last approach is used by default in Pony and you don’t need to add anything else explicitly.


Transaction isolation levels and database peculiarities
-------------------------------------------------------

See the :ref:`API Reference <transaction_isolation_levels>` for more details on this topic.


Handling disruptions
--------------------

On ``db.bind(...)`` Pony opens a connection to the database and then stores it in a thread-local connection pool.

When application code enters a database session (i.e. ``with db_session:`` context manager, or ``@db_session`` decorator), and makes a query, Pony takes any already opened connection from the pool and uses it. After exiting the database session, the connection is returned to the pool. If you enable logging, you will see a ``RELEASE CONNECTION`` message from Pony when this happens. This means that the connection was not closed, but was returned to the connection pool.

Sometimes a connection is closed by the database server. For example, when the database server is restarted. After that, any previously opened connection becomes invalid. If such a disconnect happens, it will most probably occure between database sessions, but sometimes it may happens during an active database session. Pony is prepared for such situations, and will reconnect to the database in an intelligent matter.

.. only:: comment
    Where it says "just started", is that a time-based
    check, or does it check to see if a transaction was started?
    If ``time.sleep()`` is called in the db_session context, will
    the transaction be marked as invalid?

.. only:: comment
    Which exception does pony throw when a db_session
    becomes invalid?

If Pony attempts to execute a query, but receives an error because the connection was closed, it checks the state of database session to determine if any updates were already sent to the database during the current database session. If the database session was just started, or all queries were just SELECTs, Pony assumes it is safe to re-open the connection, and continues the same database session without interrupting the application code. But if some updates were already sent to the database during this session, and then the connection is closed, those updates were lost, and it is impossible to continue the database session. In this case, Pony throws an exception.

But in most cases Pony is able to reconnect silently without disrupting application code.

.. only:: comment
    Can the link to Database.disonnect() be given with the link text "db.disconnect()",
    as opposed to parenthetically?

If you want to close a connection that is stored in the connection pool, you can call ``db.disconnect()`` (see :py:meth:`~Database.disconnect`). In multi-threaded applications this needs to be done in each thread separately.
