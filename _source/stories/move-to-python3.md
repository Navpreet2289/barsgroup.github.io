<!-- 
.. title: Рекомендации к переходу на Python 3
.. slug: move-to-python3
.. date: 2014/03/18 17:59:56
.. tags: 
.. link: 
.. description: 
.. type: text
-->

# Содержание

1. [Цель руководства](#purpose-of-the-manual)
2. [О используемых библиотеках](#about-libriries)
3. [Основные отличия Python 3.x от 2.х](#common-differences)
    - [print](#print)
    - [dict](#dict)
    - [map, filter](#map-filter)
    - [range, xrange (zip, izip)](#range-xrange)
    - [<, <=, >=, >](#operators-more-than)
    - [sorted(), list.sort()](#sorted-list-sort)
    - [cmp](#cmp)
    - [int, long](#int-long)
    - [Оператор "/"](#slash)
    -  [unicode, str, bytes](#unicode)
    -  [Относительные импорты](#relative-imports)
    -  [Метаклассы](#metaclass)
    -  [callable](#callable)
    -  [exception](#exceptions)
    -  [Перемещенные модули](#moved-modules)
4. [Полезные ссылки](#useful-links)

# <a name="purpose-of-the-manual">Цель руководства</a>
Руководство является рекомендацией по переносу кода на версию языка Python 3.x .
В руководстве приведены основные способы и приемы, используемые при портировании проектов.

**Цели руководства**:
предоставить советы и рекомендации по изменению кода для его одинакового выполнения
разными версиями интерпретатора Python (2.6, 2.7, 3.2, 3.3);


# <a name="about-libriries">О используемых библиотеках</a>

Для наиболее легкого написания кода одинаково работающего как в Python 2.6, 2.7 так и в Python 3.2, 3.3
рекомендуется использовать бибилотеку [six](https://pypi.python.org/pypi/six).

В ней собрано большее количество инструментов позволяющих писать кросверсионный код для Python 2.x-3.x.
Для нахождения мест подлежащих изменению рекомендуется воспользоваться утилитами
[2to3](http://docs.python.org/2.7/library/2to3.html) или [python-modernize](https://github.com/mitsuhiko/python-modernize).



# <a name="common-differences">Основные отличия Python 3.x от 2.х</a>

## <a name="print">print</a>

Выражение _print_ заменено функцией _print()_:

**python 2.x**:

    ::python
    print 'something'
    print 'something', #  Запятая в конце - без перехода на новую строку
    print >>sys.stderr, 'fatal error'

**python 3.x**:

    ::python
    print('something')
    print('something', end='') #  end - keyword-аргумент
    print('fatal error', file=sys.stderr)

**Что делать:**

- Каждый модуль, где используется _print_ добавить импорт _print_function_,
что позволит в Python 2.x использовать новую функцию _print_


        ::python

        from __future__ import print_function


- Заменить выражение _print_ на вызов функции _print_function_


## <a name="dict">dict</a>

Методы _items()_, _values()_, _keys()_ у объектов _dict_ теперь возвращают не список, а итерируемый объект.
Поэтому необходимо следить за результатом вызовов этих методов, например:

**python 2.x**:

    ::python
    list_ = dict_.items()
    list_.sort()

**python 3.x**:

    ::python
    list_ = sorted(dict_.items())

Также методы _iterkeys()_, _iteritems()_ and _itervalues()_ теперь не поддерживаются в python 3.x.
Так же нет метода _has_key_ в объектах словаря.

**Что делать:**

- В случаях, где используются методы списка:
    - заменить их на функции как в случае с методом списка _sort_ и функцией _sorted_
    - создать список из итератора явным вызовом функции _list_

- В местах использования результатов этих функций как итерируемых объектов
(как при проходе по ним с помощью цикла for item in d.items()) оставить всё как есть

- Заменить вызов метода _has_key_  на выражение(инструкцию) _in_.

- При создании классов, объекты которого будут иметь интерфейс объектов _dict_,
вместо  метода _has_key_ описывать метод *\_\_contains\_\_*


## <a name="map-filter">map, filter</a>

_map()_ и _filter()_ возвращают итераторы в python 3.x.
Теперь нельзя использовать _map()_ для side-эффектов функций не возвращающих результат.
Если передаваемые последовательности не одинаковой длины _map()_ остановит обработку на кратчайшей из последовательностей.

**python 2.x**:

    ::python

    >>> map(lambda x: x, [1,2,3,4])
    [1,2,3,4]

    >>> a = []
    >>> map(a.append, [1,2,3,4])
    [None, None, None, None]
    >>> a
    [1,2,3,4]

    >>> for item in map(lambda x, y: (x, y), [1,2,3,4], [1,2]):
    ... 	print(item)
    (1, 1)
    (2, 2)
    (3, None)
    (4, None)

**python 3.x**:

    ::python
    >>> map(lambda x: x, [1,2,3,4])
    <map object at 0xb6e99a8c>

    >>> a = []
    >>> map(a.append, [1,2,3,4])
    <map object at 0xb6e99acc>
    >>> a
    []

    >>> for item in map(lambda x, y: (x, y), [1,2,3,4], [1,2]):
    ... 	print(item)
    (1, 1)
    (2, 2)


**Что делать:**

- Если в этом месте действительно нужен список, то самый быстрый способ исправить это - обернуть в _list()_:

        ::python
        list(map(...))

    Однако лучше использовать списковые выражения или переписать код так чтобы не нуждаться в списке вообще.:

        ::python
        # Было
        map(lambda x:x + 1, [1,2,3,4])

        # Стало
        [x + 1 for x in [1,2,3,4]]

- Не использовать _map_ для side-эффектов функций. Вместо этого корректнее использовать цикл for:

        ::python
        for item in [1,2,3,4]:
            a.append(item)

- Честно? Кто-то пользовался тем, что _map()_ добавляет _None_ для коротких последовательностей?

## <a name="range-xrange">range, xrange (zip, izip)</a>

_xrange_ и _itertools.izip_ (Lazy-реализация _range_ и _zip_, соответственно, возвращающая итератор вместо списка)
теперь называется _range_ и _zip_, старый _range_/_zip_ - отменён за ненадобностью.

**python 2.x**:

    ::python
    >>> range(4) #  список
    [0, 1, 2, 3]

    >>> xrange(4) #  итератор
    xrange(4)

    >>> zip([1,2],[3,4]) #  список
    [(1, 3), (2, 4)]

    >>> itertools.izip([1,2],[3,4]) #  итератор
    <itertools.izip object at 0x911194c>

**python 3.x**:

    ::python
    >>> range(4) #  итератор
    range(0, 4)

    >>> xrange(4)
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    NameError: name 'xrange' is not defined

    >>> zip([1,2],[3,4]) #  итератор
    <zip object at 0xb6ec9aec>

    >>> itertools.izip([1,2],[3,4])
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    AttributeError: 'module' object has no attribute 'izip'

**Что делать:**

- Если нет особой необходимости в использовании lazy реализации _xrange_/_itertools.izip_,
то достаточно заменить везде _xrange_ на _range_.

- Если такая необходимость есть (например при больших числах),
тогда необходимо импортировать _xrange_/_zip_ из бибилиотеки _six_:

        ::python
        from six.moves import xrange
        from six.moves import zip

- Если результатом должен быть обязательно список, то можно обернуть в _list()_:

        ::python
        list(range(4))
        list(zip(...))


## <a name="operators-more-than"><, <=, >=, ></a>

Операторы сравнения теперь возбуждают исключение _TypeError_ при сравнении несравнимых типов

**python 2.x**:

    ::python
    >>> 1 > None
    True

**python 3.x**:

    ::python
    >>> 1 > None
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: unorderable types: int() > NoneType()

**Что делать:**

- Не сравнивать! Если программа работает только из-за того что числа всегда больше чем _None_, то это странная программа!

## <a name="sorted-list-sort">sorted(), list.sort()</a>

Не поддерживают аргумент cmp.

**Что делать:**

- Переписать код с использованием аргумента _key_
- Или использовать функцию _functools.cmp_to_key_

        ::python
        sorted(iterable, key=cmp_to_key(locale.strcoll))

## <a name="cmp">cmp</a>

Функции _cmp()_ в python 3.x нет. Специальный метод *\_\_cmp\_\_()* перестал использоваться.

**Что делать:**

- Вместо функции _cmp()_ можно использовать выражение _(a > b) - (a < b)_
- Вместо метода *\_\_cmp\_\_()* нужно использовать метод *\_\_lt\_\_()* для сортировки

## <a name="int-long">int, long</a>

Тип _long_ теперь называется _int_.

**Что делать:**

- Не использовать литерал L в конце чисел
- В некоторых местах пригодится _integer_types_ из библиотеки _six_

    **python 2.x**:

        ::python
        >>> isinstance(10, (int, long))
        True

    **python 3.x**:

        ::python
        >>> isinstance(10, six.integer_types)
        True

## <a name="slash">Оператор “/”</a>

Оператор / теперь всегда возвращает _float_ независимо от типов операндов

**Что делать:**

- Если нужно целочисленное деление то использовать оператор //
- Для использования в Python 2.x нового оператора деления добавить в импорт _division_

        ::python
        from __future__ import division

## <a name="unicode">unicode, str, bytes</a>

Теперь есть только один строковый тип - _str_,
схожий по поведению с типом _unicode_ из Python 2.x.
Базового типа для строк _basestring_ в python 3 не существует.
Все строки превращаются в юникодные. Литерал _u_ для юникодных строк теперь не обязателен
(Его добавили только в версии python 3.3, как часть обратной совместимости).

**Что делать:**


- Для сохранения совместимости с Python 2.x заменить _str_ и _unicode_ на _text_type_ из библиотеки _six_.
_text_type_ будет равен _unicode_ для Python 2.x и _str_ для Python 3.x

- Так же заменить _basestring_ на _string_types_ из библиотеки _six_.
_string_types_ равен _(basestring,)_ для Python 2.x и _(str, )_ для Python 3.x


## <a name="relative-imports">Относительный импорт</a>

Теперь импорты модулей находящихся в том же пакете,
что и модуль откуда производиться импоритрование должны начинаться с точки.

**python 2.x**:

    ::python
    >>> from models import BaseModel
    >>> import models

**python 3.x**:

    ::python
    >>> from .models import BaseModel
    >>> from . import models

**Что делать:**

- Переделать все относительные импорты на _from ._

## <a name="metaclass">Метаклассы</a>

Изменился синтаксис задания метакласса классу

**python 2.x**:

    ::python
    class C(A):
        __metaclass__ = M

**python 3.x**:

    ::python
    class C(A, metaclass=M):
        pass

**Что делать:**

- Заменить атрибут класса *\_\_metaclass\_\_* на наследование от результата функции _with_metaclass_ из библиотеки _six_

    Конструкцию вида:

        ::python
        class C(A, B):
             __metaclass__ = M

    Заменить на:

        ::python
        class C(six.with_metaclass(M, A), B):

## <a name="callable">callable</a>

Функция callable удалена в python 3.x.

**python 2.x**:

    ::python
    >>> callable(map)
            True

**python 3.x**:

    ::python
    >>> isinstance(map, collections.Callable)
        True

**Что делать:**

- Использовать функцию callable из библиотеки six

        ::python
        six.callable(map)


## <a name="exceptions">Исключения</a>

В python 3.x синтаксис обработки исключений типа except <Exception>, err: - запрещён.
Вместо запятой обязательно использовать ключевое слово _as_.

**python 2.x**:

	::python
	try:
		a=b/c
	except ZeroDivisionError, err:
		print err

**python 3.x**:

	::python
	try:
		a=b/c
	except ZeroDivisionError as err:
		print err

**Что делать:**

- Использовать везде только новый синтаксис обработки исключений


## <a name="moved-modules">Перемещенные модули</a>
В Python 3.x были перемещены/переименованны некоторые модули.

**python 2.x**:

    ::python
    from cStringIO import StringIO

**python 3.x**:

    ::python
    from io import StringIO

**Что делать:**

- Для совместимости с Python 2.x эти модули можно импортировать из библиотеки six

	    ::python
	    from six.moves import cStringIO



## <a name="useful-links">Полезные ссылки</a>

[http://docs.python.org/3.0/whatsnew/3.0.html](http://docs.python.org/3.0/whatsnew/3.0.html)

[http://docs.python.org/dev/howto/pyporting](http://docs.python.org/dev/howto/pyporting)

[http://python3porting.com/](http://python3porting.com/)

[https://docs.djangoproject.com/en/dev/topics/python3/](https://docs.djangoproject.com/en/dev/topics/python3/)

[http://pythonhosted.org/six/](http://pythonhosted.org/six/)
