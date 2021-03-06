.. _thingflow-application:

5. ThingFlow Application
========================
Now that our board is up and running, we can use ThingFlow to build a simple
application.

ThingFlow
---------
ThingFlow is a Python 3 framework for building IoT event processing and
filtering applications. It is centered around *event streams*, which are
infinite sequences of timestamped sensor value samples. The key components
in ThingFlow are *sensors*, which capture data from the outside world,
*things*, which transform event streams, and the *scheduler* which manages
the recurring sampling of sensors. ThingFlow allows you to write your
IoT code as data flows ("from this to that") instead of as procedural
code ("if this then that").

There are two implementations of ThingFlow in Python. The goal is to
provide a common API from very small sensor nodes (like the ESP8266) to
large servers (for event concentration and data science). We will be using
the MicroPython version, which may be found in the ``micropython`` subdirectory
of the `thingflow-python <https://github.com/mpi-sws-rse/thingflow-python>`_
repository.

Due to the memory limitations of the ESP8266, the MicroPython version is
very stripped down, and only supports a subset of the full ThingFlow API.
That will be fine for our application, as the focus of our code will be to
sample the light sensor and send the values over the network to a larger
system for processing and / or storage.

Reading the Light Sensor
------------------------
A *sensor* in ThingFlow is a Python object which satisfies two
criteria:

1. It provides a ``sensor_id`` property which can be used in event
   messages to uniquely identify the sensor.
2. It provides a ``sample()`` method which returns the current value
   associated with the sensor.

The ThingFlow MicroPython implementation includes a sensor implementation
for the TSL2591 lux sensor. We will first copy this over to the ESP8266
and verify that we can read the sensor.

Copying Files
~~~~~~~~~~~~~
Close any previous ``screen`` sessions with your ESP8266 system, and
restart it.

In a terminal session, go to the ``micropython`` subdirectory of the
ThingFlow repository. From this directory, run the ``mpfshell``
utility. In this session type the following (substituting ``tty.SLAB_USBtoUART``
or ``ttyUSB0`` for TTYDEVICE)::

  open TTYDEVICE
  ls

The ``open`` command establishes a connection to your ESP8266. The ``ls`` will
list the files on that system (MicroPython includes a very simple filesystem
implementation). Initially, you should only see the file ``boot.py``, which is
installed as a part of the firmware flash.

Now, run the command ``lls``. This lists the files on your host system, in the
directory where you ran ``mpfshell``. Now, run the following::

  lcd sensors
  lls

This changes the host directory to the ``sensors`` subdirectory and lists the
files. One of the files you should see is ``tsl2591.py``. This is the code which
implements the interface to the light sensor.

Now, run::

  put tsl2591.py
  ls

This copies the file ``tsl2591.py`` to the ESP8266 and lists the files on it.
We should now see our sensor code file, in addition to ``boot.py``.

Calling the Light Sensor
~~~~~~~~~~~~~~~~~~~~~~~~
Next, we want to import and call the sensor code from the MicroPython REPL.
There are two ways to do this:

1. You can access the REPL directly from ``mpfshell``. Just enter the command
   ``repl``, and you will be in the REPL. You can later exit the REPL and get
   back to the main shell via the key combination CONTROL and "]".
2. Alternatively, you can exit ``mpfshell`` and then run ``screen`` again to get
   a REPL directly.

Either way, once we are in the REPL, we want to import the Tsl2591 class from
``tsl2591.py``, instantiate an instance, and call its ``sample()`` method.
Here is an example session:

.. code-block:: python

  >> from tsl2591 import Tsl2591
  >>> tsl = Tsl2591('lux-1')
  >>> tsl.sample()
  296.3712

The ``lux-1`` passed to the the ``Tsl2591`` constructor is the sensor id. In
this case, it can be an arbitrary string. The call to ``sample()`` may take
almost a second to complete (it takes some time to properly sample the value
from the lux sensor). The value returned in the light reading in units of
*lux*. Try running the sample call again with your hand covering the light
sensor -- it should return a lower value. Now, we have verified that the light
sensor is working for us!

Debugging problems
~~~~~~~~~~~~~~~~~~
If you get an error when instantiating or reading from the light sensor,
you might have a bad connection or a problem with your sensor. Here's what
an error might look like in your REPL session:

.. code-block:: python

    >>> from tsl2591 import Tsl2591
    >>> tsl = Tsl2591('lux-1')
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "tsl2591.py", line 83, in __init__
      File "tsl2591.py", line 91, in set_timing
      File "tsl2591.py", line 150, in enable
      File "tsl2591.py", line 61, in write_byte_data
    OSError: [Errno 19] ENODEV

If you get an error like this, double-check your wiring.
If you believe all your connections are correct,
you might check whether one of the physical connections is bad. Set your
multitester to resistance mode. For each connection, place one
lead on the pin of the ESP2866 board and the other on the associated pin of
the TSL2591 board. The resistance should measure zero if there is indeed
a connection.

A Light Sampling Application
----------------------------
Now, we will copy over the main module of ThingFlow and use the scheduler
to periodically call our sample method and print the result. First, start
``mpfshell`` in the ``micropython`` directory. Copy ``thingflow.py`` over
to the ESP8266 as follows::

  open TTYDEVICE
  put thingflow.py
  ls

You should see that ``thingflow.py`` is now on the ESP8266.

Next, go back to the MicroPython REPL. We will import the ThingFlow core and
our sensor. Then we will instantiate the sensor object and a scheduler. Finally,
we will call the scheduler with the sensor, asking it to sample the sensor
once every two seconds and print the resulting event. Here is the REPL session:

.. code-block:: python

    >>> from thingflow import *
    >>> from tsl2591 import Tsl2591
    >>> tsl = Tsl2591('lux-1')
    >>> sched = Scheduler()
    >>> sched.schedule_sensor(tsl, 2.0, Output())
    <closure>
    >>> sched.run_forever()
    ('lux-1', 344, 294.9023)
    ('lux-1', 345, 294.9023)
    ('lux-1', 347, 294.9023)
    ('lux-1', 349, 288.2113)
    ('lux-1', 351, 245.6161)
    ('lux-1', 352, 214.1184)
    ('lux-1', 354, 48.14401)
    ('lux-1', 356, 50.75521)
    ('lux-1', 358, 294.9023)

The ``schedule_sensor()`` call takes three parameters: the sensor
object to be schedule, the sample interval in seconds, and the downstream
data flow. In this case we are just calling the ``Output`` "thing" to
print the messages.

The tuples being printed have three elements: the sensor id, a timestamp,
and the sensor reading.

In the :ref:`next section <mqtt>`,
we'll see how we can get these samples off the ESP2866
using its WiFi radio and the MQTT protocol.
