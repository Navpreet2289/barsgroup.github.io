<!-- 
.. title: Переход на концепцию m3 2.0
.. slug: move-to-m3-20
.. date: 2013/06/25 17:42:44
.. tags: 
.. link: 
.. description: 
.. type: text
-->


По итогам svg-шек, grep-ов и прочей магии получили следующую картину по рефакторингу пакета m3:

- core
    + registry - Пока просто выделяется в отдельный [контриб](https://bitbucket.org/barsgroup/registry).
    Следующие версии предполагают безумный рефакторинг
    + history - Нигде не используется и удаляется
    + middleware.py - Выкидывается все, кроме AutoLogout и помещается в core/__init__.py
    + json - Encoder пока оставляем, переносится в core/__init__.py
    + exceptions - Удаляется, AppLogicException перемещается в __init__.py
    + plugins - Перемещается в [m3_legacy](https://bitbucket.org/barsgroup/m3-legacy)
    + excel_reporting - Перемещен в отдельный [контриб](https://bitbucket.org/barsgroup/excel-reporting)
    + notifications - Нигде не используется и удаляется
    + replication - Нигде не используется и удаляется
- contrib
    + setup_environ - Удаляется, нужен был для встраивания prepare_env.py
    + designer - Там лежит management-команда для запуска дизайнера, переезжает в сам контриб дизайнера
data
    + mie - Удаляется. Используется совсем немного в Школах.
    + caching - Переносится в core
    + proxy - Нигде не используется и удаляется
db
    + mptt_util - Внутри находится механизм перестройки mptt-дерева, пока непонятно куда вынести.
    Использовать последнюю версию [mptt](https://github.com/django-mptt/django-mptt)
    + alchemy_wrapper - Используется в report-generator'e и ЭПК.  Из ЭПК выпиливается и переносится в report-generator
    + ddl - Злостно удаляется
    + \__init__ - Частично внутренности этого модуля переедут в [m3_legacy](https://bitbucket.org/barsgroup/m3-legacy)
    после более глубокого исследования
    + api - Перемещено в [m3_legacy](https://bitbucket.org/barsgroup/m3-legacy)
helpers
    + logger - Будет перемещен в [m3_legacy](https://bitbucket.org/barsgroup/m3-legacy)
    и по-возможности будет использоваться внутри себя django-подход
    +  loader - Перемещается в [m3_legacy](https://bitbucket.org/barsgroup/m3-legacy)
    + urls - Перемещается в core/actions.py
    + ui - Будет удален. Так же будет выпелен из ui/actions/results.py, из m3_users и Родплаты
    + validation - Перемещается в m3_legacy
    + queries - Будет удален. Проекты, которые используют данный модуль - m3_contragents, Род. Плата, Закупки
    + Содержимое __init__.py - Перемещается в [m3_legacy](https://bitbucket.org/barsgroup/m3-legacy)
    + icons - в пакет с [m3-ext](https://bitbucket.org/barsgroup/m3-ext)
    + js - Будет удален
    + datastructures - Удаляется
    + datagrouping, datagrouping_alchemy - Переносится в отдельный пакет. И рефакториться в следующих версиях модуля.
    + config - Будет удален
    + checksum - Будет перенесен в отдельный пакет
    + users - BaseUserProfile выкидывается, декоратор authenticated_user_required перенесется в core
    + autonum - Нигде не используется, будет удален
    + sqlite - Будет удален
    + mocks - Будет удален
- misc - Удаляется
- templatetags - Единственная функция переедет в core
- ui/actions/packs и ui/actions/dicts - Переносятся в отдельный пакет, например, simple_dict
- m3_workflow - Удаляется
- ui вместе со статикой - переносится в отдельный пакет [m3-ext](https://bitbucket.org/barsgroup/m3-ext)
- ui/actions/ переедет в core
- ui/actions/gears, ui/actions/app_ui - переедет в [m3-ext](https://bitbucket.org/barsgroup/m3-ext)

[m3_legacy](https://bitbucket.org/barsgroup/m3-legacy) - Пакет, который будет содержать устаревший код,
в нем будут исправляться только ошибки, которых и так минимальное количество. Этот контриб далее поддерживаться не будет.

Все, что останется войдет в сборку m3 2.0. Эта версия будет содержать нужный код, который просто изменил текущее местоположение,
рефакторинг будет производиться в следующих версиях.

Так же для перехода на новую версию подготовлен [скрипт](https://gist.github.com/prefer/7e831f6909afcdfc814e)
для понимания какие импорты сломались и как их починить.