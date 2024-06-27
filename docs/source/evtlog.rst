Event Log
=========

.. module:: windows.winobject.event_log


Some part of the Event Log WINAPI are not straightforward.

I have tried to offer some abstraction without completly hidding the some underlying subtilities (for now).

The current API may need some works to provide simpler/highter level API in the future.

The :class:`EvtlogManager` instance is accessible via :py:attr:`windows.system.event_log
<windows.winobject.system.System.event_log>`

For now, the best thing to do is look at the sample:

.. note::

    See sample :ref:`sample_event_log`


EvtlogManager
"""""""""""""

.. autoclass:: EvtlogManager
    :special-members: __getitem__


Channel
"""""""

EvtChannel
''''''''''

.. autoclass:: EvtChannel

ChannelConfig
'''''''''''''

.. autoclass:: ChannelConfig


Publisher
"""""""""

EvtPublisher
''''''''''''

.. autoclass:: EvtPublisher

PublisherMetadata
'''''''''''''''''

.. autoclass:: PublisherMetadata


EvtFile
"""""""

.. autoclass:: EvtFile


Event
"""""

EvtEvent
''''''''

.. autoclass:: EvtEvent

EventMetadata
'''''''''''''

.. autoclass:: EventMetadata


EvtQuery
''''''''

.. autoclass:: EvtQuery



TODO
""""

.. autoclass:: PropertyArray



