Dooming Transactions
====================

A doomed transaction behaves exactly the same way as an active transaction but
raises an error on any attempt to commit it, thus forcing an abort.

Doom is useful in places where abort is unsafe and an exception cannot be
raised.  This occurs when the programmer wants the code following the doom to
run but not commit. It is unsafe to abort in these circumstances as a following
get() may implicitly open a new transaction.

Any attempt to commit a doomed transaction will raise a DoomedTransaction
exception.

An example of such a use case can be found in
zope/app/form/browser/editview.py.  Here a form validation failure must doom
the transaction as committing the transaction may have side-effects. However,
the form code must continue to calculate a form containing the error messages
to return.

For Zope in general, code running within a request should always doom
transactions rather than aborting them. It is the responsibilty of the
publication to either abort() or commit() the transaction. Application code can
use savepoints and doom() safely.

To see how it works we first need to create a stub data manager:

.. doctest::

    >>> from transaction.interfaces import IDataManager
    >>> from zope.interface import implementer
    >>> @implementer(IDataManager)
    ... class DataManager:
    ...     def __init__(self):
    ...         self.attr_counter = {}
    ...     def __getattr__(self, name):
    ...         def f(transaction):
    ...             self.attr_counter[name] = self.attr_counter.get(name, 0) + 1
    ...         return f
    ...     def total(self):
    ...         count = 0
    ...         for access_count in self.attr_counter.values():
    ...             count += access_count
    ...         return count
    ...     def sortKey(self):
    ...         return '1'

Start a new transaction:

.. doctest::

    >>> import transaction
    >>> txn = transaction.begin()
    >>> dm = DataManager()
    >>> txn.join(dm)

We can ask a transaction if it is doomed to avoid expensive operations. An
example of a use case is an object-relational mapper where a pre-commit hook
sends all outstanding SQL to a relational database for objects changed during
the transaction. This expensive operation is not necessary if the transaction
has been doomed. A non-doomed transaction should return False:

.. doctest::

    >>> txn.isDoomed()
    False

We can doom a transaction by calling .doom() on it:

.. doctest::

    >>> txn.doom()
    >>> txn.isDoomed()
    True

We can doom it again if we like:

.. doctest::

    >>> txn.doom()

The data manager is unchanged at this point:

.. doctest::

    >>> dm.total()
    0

Attempting to commit a doomed transaction any number of times raises a
DoomedTransaction:

.. doctest::

    >>> txn.commit() # doctest: +IGNORE_EXCEPTION_DETAIL
    Traceback (most recent call last):
    DoomedTransaction: transaction doomed, cannot commit
    >>> txn.commit() # doctest: +IGNORE_EXCEPTION_DETAIL
    Traceback (most recent call last):
    DoomedTransaction: transaction doomed, cannot commit

But still leaves the data manager unchanged:

.. doctest::

    >>> dm.total()
    0

But the doomed transaction can be aborted:

.. doctest::

    >>> txn.abort()

Which aborts the data manager:

.. doctest::

    >>> dm.total()
    1
    >>> dm.attr_counter['abort']
    1

Dooming the current transaction can also be done directly from the transaction
module. We can also begin a new transaction directly after dooming the old one:

.. doctest::

    >>> txn = transaction.begin()
    >>> transaction.isDoomed()
    False
    >>> transaction.doom()
    >>> transaction.isDoomed()
    True
    >>> txn = transaction.begin()

After committing a transaction we get an assertion error if we try to doom the
transaction. This could be made more specific, but trying to doom a transaction
after it's been committed is probably a programming error:

.. doctest::

    >>> txn = transaction.begin()
    >>> txn.commit()
    >>> txn.doom()
    Traceback (most recent call last):
        ...
    ValueError: non-doomable

A doomed transaction should act the same as an active transaction, so we should
be able to join it:

.. doctest::

    >>> txn = transaction.begin()
    >>> txn.doom()
    >>> dm2 = DataManager()
    >>> txn.join(dm2)

Clean up:

.. doctest::

    >>> txn = transaction.begin()
    >>> txn.abort()
