.. PyZMQ eventloop doc, by Min Ragan-Kelley, 2011

.. _eventloop:

Tornado Eventloop with PyZMQ
============================

Facebook's `Tornado`_ includes an eventloop for handing poll events on filedescriptors and
native sockets. We have included a small part of Tornado (specifically its
:mod:`.ioloop`), and adapted its :class:`IOStream` class into :class:`.ZMQStream` for
handling poll events on ØMQ sockets. A ZMQStream object works much like a Socket object,
but instead of calling :meth:`~.Socket.recv` directly, you register a callback with
:meth:`~.ZMQStream.on_recv`. callbacks can also be registered for send and error events
with :meth:`~.ZMQStream.on_send` and :meth:`~.ZMQStream.on_err` respectively.


:func:`install()`
-----------------

With PyZMQ's ioloop, you can use zmq sockets in any tornado application.  You must first
install PyZMQ's :class:`.IOLoop`, with the :func:`.ioloop.install` function:

.. sourcecode:: python

    from zmq.eventloop import ioloop
    ioloop.install()

This sets the global instance of :class:`tornado.ioloop.IOLoop` with the global instance of
our IOLoop class. The reason this must happen is that tornado objects avoid having to pass
the active IOLoop instance around by having a staticmethod :meth:`.IOLoop.instance`, which
always returns the active instance. If PyZMQ's IOLoop is installed after the first call to
:meth:`.IOLoop.instance()` (called in almost every tornado object constructor), then it will
raise an :exc:`AssertionError`, because the global IOLoop instance has already been
created, and proceeding would result in not all objects being associated with the right
IOLoop.

It is possible to use PyZMQ sockets with tornado *without* calling :func:`.ioloop.install`,
but it is less convenient. First, you must instruct the tornado IOLoop to use the zmq poller:

.. sourcecode:: python

    from tornado.ioloop import IOLoop
    from zmq.eventloop.ioloop import ZMQPoller
    
    loop = IOLoop(ZMQPoller())

Then, when you instantiate tornado and ZMQStream objects, you must pass the `io_loop`
argument to ensure that they use this loop, instead of the global instance.  You can
install this IOLoop as the global tornado instance, with:

.. sourcecode:: python

    loop.install()

but it will **NOT** be the global *pyzmq* IOLoop instance, so it must still be passed to
your ZMQStream constructors.


:meth:`send`
------------

ZMQStream objects do have :meth:`~.ZMQStream.send` and :meth:`~.ZMQStream.send_multipart`
methods, which behaves the same way as :meth:`.Socket.send`, but instead of sending right
away, the :class:`.IOLoop` will wait until socket is able to send (for instance if ``HWM``
is met, or a ``REQ/REP`` pattern prohibits sending at a certain point). Messages sent via
send will also be passed to the callback registered with :meth:`~.ZMQStream.on_send` after
sending.

:meth:`on_recv`
---------------

:meth:`.ZMQStream.on_recv` is the primary method for using a ZMQStream. It registers a
callback to fire with messages as they are received, which will *always* be multipart,
even if its length is 1. You can easily use this to build things like an echo socket:

.. sourcecode:: python

    s = ctx.socket(zmq.REP)
    s.bind('tcp://localhost:12345')
    loop = ioloop.IOLoop.instance()
    stream = ZMQStream(s, loop)
    def echo(msg):
        stream.send_multipart(msg)
    stream.on_recv(echo)
    loop.start()

on_recv can also take a `copy` flag, just like :meth:`.Socket.recv`. If `copy=False`, then
callbacks registered with on_recv will receive tracked :class:`.Message` objects instead of
bytes.

:meth:`flush`
-------------

Sometimes with an eventloop, there can be multiple events ready on a single iteration of
the loop. The :meth:`~.ZMQStream.flush` method allows developers to pull messages off of
the queue to enforce some priority over the event loop ordering. flush pulls any pending
events off of the queue. You can specify to flush only recv events, only send events, or
any events, and you can specify a limit for how many events to flush in order to prevent
starvation.

.. _Tornado: https://github.com/facebook/tornado