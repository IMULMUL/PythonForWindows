.. _py_windows_encoding:

Python, Windows & encoding
==========================

This part is only the result of my experimentations and readings and might not reflect the full picture.

My near-futur goal is to have PythonForWindows fully support unicode and use W (Wide) API.
This will result in most of the strings handled by PFW being unicode strings.

In python3 it should work as-is because Windows console handle Wide unicode and Py3 is using the ``ConsoleWriteW`` API directly.
The fact that python3 :class:`str` type represent unicode string helps a lot.



But, As I want to maintains Python2.7 compatibilty and easyness of use, things beging to become tricky.

The main problem is the ability to print unicode (chinese / russian / ...) in the console and object repr.

Example:
    >>> windows.crypto.Certificate.from_file("omae.cer")
    <Certificate "お前はもう死んでい" serial="19 da cc 2b a5 61 b6 98 4e 0d 6c 0c cb ce e6 99">

To achieve this in Py2 the required additionnal configuration of the console should be::

    set PYTHONIOENCODING=utf-8 # Set environnement variable PYTHONIOENCODING to utf-8. Allowing UTF8 encoding for python output (including sys.stdout).
    chcp 65001 # Setting the console code page to 65001 (UTF-8) Allowing the console to print the correct chars when receiving UTF-8 data


For it to work optimaly in Py2.7, it require some tricks in both the codebase, the python configuration and console configuration.

My goal is to offer the better experience even without any Python/Console configuration.
But with the additionnal configuration it should be able to print anything without trouble.

What does this additional configuration does ?
''''''''''''''''''''''''''''''''''''''''''''''


PYTHONIOENCODING
^^^^^^^^^^^^^^^^


As the official `Python2 documentation <https://docs.python.org/2/using/cmdline.html#envvar-PYTHONIOENCODING>`_ says::

    Overrides the encoding used for stdin/stdout/stderr, in the syntax encodingname:errorhandler. The :errorhandler part is optional and has the same meaning as in str.encode().

The goal of setting this environnement variable is to have the stdout encoding set to ``utf-8``.
This will allow a somewhat seamless encoding of any unicode string you try to print.


By default, the ``sys.stdout.encoding`` is set according to the current console output codepage (`GetConsoleOutputCP <https://docs.microsoft.com/en-us/windows/console/getconsoleoutputcp>`_) at python initialisation.
This can be seen in `Py_InitializeEx <https://github.com/python/cpython/blob/2.7/Python/pythonrun.c#L354>`_.

Most of the default codepage (437 in my case) cannot encode all unicode character.
These two elements together are the reason why priting an unicode string without any change in Python2 will lead to this type of problem::


    $ chcp
    Active code page: 437

    $ python -s -E
    Python 2.7.12 (v2.7.12:d33e0cf91556, Jun 27 2016, 15:19:22) [MSC v.1500 32 bit (Intel)] on win32
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import sys
    >>> sys.stdout.encoding
    'cp437'
    >>> print(u"\u304a\u524d\u306f\u3082\u3046\u6b7b\u3093\u3067\u3044")
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "C:\Python27\lib\encodings\cp437.py", line 12, in encode
        return codecs.charmap_encode(input,errors,encoding_map)
    UnicodeEncodeError: 'charmap' codec can't encode characters in position 0-8: character maps to <undefined>


.. note::

    Note that the fails is in ``cp437.py`` directly linked to ``sys.stdout.encoding``


By setting ``PYTHONIOENCODING`` you force the ``sys.stdout.encoding``::

    $ chcp
    Active code page: 437

    $ set PYTHONIOENCODING=utf-8

    $ python
    Python 2.7.12 (v2.7.12:d33e0cf91556, Jun 27 2016, 15:19:22) [MSC v.1500 32 bit (Intel)] on win32
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import sys
    >>> sys.stdout.encoding
    'utf-8'
    >>> print(u"\u304a\u524d\u306f\u3082\u3046\u6b7b\u3093\u3067\u3044")
    πüèσëìπü»πééπüåµ¡╗πéôπüºπüä # Gibberish due to bad codepage in console


Well, it does not raise an exception anymore but it's printing gibberish.
This is because, as-is the console still expects cp437 as an output.


chcp 65001
^^^^^^^^^^

The `chcp <https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/chcp>`_ commande allow to display/change the active console code page.
The codepage 65001 stand for UTF-8.

Thus, setting the console code page to 65001 will tell it to expect UTF-8 as a program output. Which is perfect with our previous setup of ``PYTHONIOENCODING``::

    $ set PYTHONIOENCODING=utf-8

    $ chcp 65001
    Active code page: 65001

    $ python
    Python 2.7.12 (v2.7.12:d33e0cf91556, Jun 27 2016, 15:19:22) [MSC v.1500 32 bit (Intel)] on win32
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import sys
    >>> sys.stdout.encoding
    'utf-8'
    >>> print(u"\u304a\u524d\u306f\u3082\u3046\u6b7b\u3093\u3067\u3044")
    お前はもう死んでい


.. warning:

    Well, if ``chcp 65001`` stand for UTF-8 and that without ``PYTHONIOENCODING`` python use the code page has encoding. We do we even need to setup ``PYTHONIOENCODING`` ?

    The response is quite sad..
    Python setp stdout encoding to cp65001 but it does not recognize the cp65001 encoding as UTF-8. It does not know about it !

    See::

        $ set PYTHONIOENCODING=

        $ chcp 65001
        Active code page: 65001

        $ python
        Python 2.7.12 (v2.7.12:d33e0cf91556, Jun 27 2016, 15:19:22) [MSC v.1500 32 bit (Intel)] on win32
        Type "help", "copyright", "credits" or "license" for more information.

        >>> import sys

        LookupError: unknown encoding: cp65001 # Broken Interactive console :(

        $ python -c "import sys; print(sys.stdout.encoding);  print(u'\u304a\u524d\u306f\u3082\u3046\u6b7b\u3093\u3067\u3044')"
        cp65001
        Traceback (most recent call last):
        File "<string>", line 1, in <module>
        LookupError: unknown encoding: cp65001


The case of __repr__
''''''''''''''''''''

The case of UTF-8 and __repr__ in Python2.7 is more tricky.
I pay particular attention to this case because I am an heavy user of the interactive console and object ``__repr__`` to explore Windows.

the ``__repr__`` function cannot return an ``unicode`` object and must return a ``str``.
But as I want to be able to output repr for objects with unicode attributes (like a お前はもう死んでい certificate), I need to encode my unicode __repr__.

In those cases, I encode the repr with the stdout encoding (with backslash escape for non-encodable characters).
This should assure that the result can always be written to stdout.

But based on the encoding of stdout and the code page 3 case may appear.
For example with an object having the following unicode repr:

    * <Certificate "お前はもう死んでい" serial="19 da cc 2b a5 61 b6 98 4e 0d 6c 0c cb ce e6 99">

The possibilities are:

    * stdout encoding do not handle full unicode (like cp437)
        * repr will be backslash escaped to allow printing
        * <Certificate "\\u304a\\u524d\\u306f\\u3082\\u3046\\u6b7b\\u3093\\u3067\\u3044" serial="19 da cc 2b a5 61 b6 98 4e 0d 6c 0c cb ce e6 99">

    * stdout encoding is utf-8 but code page is not (like cp437)
        * console will output gibberish by trying to interpret utf-8 as a custom CodePage
        * <Certificate "πüèσëìπü»πééπüåµ¡╗πéôπüºπüä" serial="19 da cc 2b a5 61 b6 98 4e 0d 6c 0c cb ce e6 99">

    * stdout encoding is utf-8 and code page is 65001
        * it works !
        * <Certificate "お前はもう死んでい" serial="19 da cc 2b a5 61 b6 98 4e 0d 6c 0c cb ce e6 99">



Sample of test
''''''''''''''

I have created a sample ``samples\encoding\check_encoding_config.py`` that should help to understand and verify the current configuration of the console.
The code check and display the values of ``PYTHONIOENCODING``, ``sys.stdout.encoding`` and the current console code page. It also tries to print an unicode string as well as an unicode object __repr__.


No setup
^^^^^^^^

``PYTHONIOENCODING`` is not set and code page is something like 437.

    * The printing of an unicode string will fail
    * The printing of an unicode __repr__ will display escaped unicode values
        * PFW make its best to not raise an encoding related exception on __repr__ by checking ``sys.stdout.encoding``

Example::

    $ chcp 437
    Active code page: 437

    $ set PYTHONIOENCODING=

    $ python samples\encoding\check_encoding_config.py
    Python version is <2.7.12 (v2.7.12:d33e0cf91556, Jun 27 2016, 15:19:22) [MSC v.1500 32 bit (Intel)]>
    Py2 python/console configuration analysis:
    [*] env[PYTHONIOENCODING] = None
        [-] No env variable <PYTHONIOENCODING>.
            sys.stdout encoding will only depends on your console codepage. Leading to high probability of EncodingError if printing unicode string
    [*] sys.stdout.encoding = cp437
        [-] Unoptimal stdout encoding
            Recommended fix is setting PYTHONIOENCODING == utf-8
    [*] Console Codepage = 437
        [-] Non UTF-8 codepage for the current console
            Setting codepage to UTF8 (chcp 65001) will ensure currect output with PYTHONIOENCODING UTF-8
    [-] Error printing unicode string: 'charmap' codec can't encode characters in position 23-31: character maps to <undefined>
    Unicode object repr: <MyUtf8Object name="\u304a\u524d\u306f\u3082\u3046\u6b7b\u3093\u3067\u3044-\u043a\u0430\u043a\u0438\u0435_\u0444\u0436\u044e\u0449\u0434\u0444\u044f">


PYTHONIOENCODING only
^^^^^^^^^^^^^^^^^^^^^

``PYTHONIOENCODING`` is set to ``utf-8`` and code page is something like 437.

    * The printing of an unicode string will work but display gibberish
    * The printing of an unicode __repr__ will work but display gibberish

This is due to a mis-match between python output encoding and the console expected output.

Example::

    $ chcp
    Active code page: 437

    $ set PYTHONIOENCODING=utf-8

    $ python samples\encoding\check_encoding_config.py
    Python version is <2.7.12 (v2.7.12:d33e0cf91556, Jun 27 2016, 15:19:22) [MSC v.1500 32 bit (Intel)]>
    Py2 python/console configuration analysis:
    [*] env[PYTHONIOENCODING] = utf-8
        [+] Optimal PYTHONIOENCODING
    [*] sys.stdout.encoding = utf-8
    [*] Console Codepage = 437
        [-] Non UTF-8 codepage for the current console
            Setting codepage to UTF8 (chcp 65001) will ensure currect output with PYTHONIOENCODING UTF-8
    Unicode string print: <πüèσëìπü»πééπüåµ¡╗πéôπüºπüä-╨║╨░╨║╨╕╨╡_╤ä╨╢╤Ä╤ë╨┤╤ä╤Å>
    Unicode object repr: <MyUtf8Object name="πüèσëìπü»πééπüåµ¡╗πéôπüºπüä-╨║╨░╨║╨╕╨╡_╤ä╨╢╤Ä╤ë╨┤╤ä╤Å">


Full setup
^^^^^^^^^^

``PYTHONIOENCODING`` is set to ``utf-8`` and code page is 65001.
Everything should work::

    $ set PYTHONIOENCODING=utf-8

    $ chcp 65001
    Active code page: 65001

    $ python samples\encoding\check_encoding_config.py
    Python version is <2.7.12 (v2.7.12:d33e0cf91556, Jun 27 2016, 15:19:22) [MSC v.1500 32 bit (Intel)]>
    Py2 python/console configuration analysis:
    [*] env[PYTHONIOENCODING] = utf-8
        [+] Optimal PYTHONIOENCODING
    [*] sys.stdout.encoding = utf-8
    [*] Console Codepage = 65001
    Unicode string print: <お前はもう死んでい-какие_фжющдфя>
    Unicode object repr: <MyUtf8Object name="お前はもう死んでい-какие_фжющдфя">