<!-- 
.. title: Комментарий к изменениям от 08-04-2014
.. slug: comment-changelog-08-04-2014
.. date: 2014/04/10 14:24:08
.. tags: 
.. link: 
.. description: 
.. type: text
-->

# Содержание

  - [Desktop View](#desktop-view)
  - [Особенности поиска паков/экшнов](#find-packs-and-actions)
  - [Overrides](#overrides)

## <a name="desktop-view">Desktop View</a>

В проектах, переходящих на **m3-ext==2.0.6** **view** для построения Рабочего Стола должна быть замена на **view**, предоставляемой платформой.

Проектный шаблон страницы Рабочего Стола, должен расширять платформенный с помощью ``{% extend 'm3_workspace.html' %}``. Для расширения доступны блоки:

- ``{% block extra_head %}`` - раширение <head></head>
- ``{% block desktop_widgets %}`` - сюда добавляются виджеты Рабочего Стола
- ``{% block extra_content %}`` - "подвал" для того, что не должно непосредственно попасть на Рабочий Стол.

### Пример

#### urls.py

    ::python
    urlpatterns = patterns('',
        ...
        url(r'^$', workspace(
            'prj_workspace.html',
            context_processors=[
                prj_context_proc1,
                prj_context_proc2
            ]
        )),
        ...
    )

Здесь ``'prj_workspace.html'`` - имя проектного шаблона Рабочего Стола. ``context_processors`` - список процессоров контекста - функций, подобных *процессорам контекста Django* (``request -> dict``).

### Пример процессора контекста

    ::python
    def prj_context_proc1(request):
        return {
            'user_name': (
                u'Аноним'
                if request.user is AnonimousUser
                else request.user.username
            ),
            'notifications': Notification.objects.filter(
                user=request.user,
                readed=False,
            ).exists()
        }

### Видимость элементов Рабочего Стола

Пункт меню/ярлык теперь **будет скрыт** от пользователя, если

- создан с помощью ``DesktopShortcut``
- **pack/action**, переданный в ``DesktopShortcut`` требует проверки прав (``need_check_permission=True``)
- пользователь не имеет прав на выполнение этого **pack/action** (``pack.has_perm(request) -> False``)

При несоблюдении любого из этих условий элемент **будет отображаться**.

Т.о. видимость элементов определяется исключительно правами и не нужно использовать условное добавление в ``app_meta.py``!

**ВАЖНО!** Метароли оставлены для управления интерфейсом на уровне view. При этом все ярлыки/пункты меню должны добавляться метароли ``GENERIC_USER``.



## <a name="find-packs-and-actions">Особенности поиска паков/экшнов</a>

В **m3-core** с версии **2.0.11** произошли измениения в механизме поиска экземпляров зарегистрированных в системе экшнов/паков.

### Controller -> ControllerCache
Теперь паки/экшны нужно искать при помощи методов ``ControllerCache.find_pack`` и ``ControllerCache.find_action``, вместо поиска в конкретных контроллерах. Поиск в контроллерах в данной версии работать будет, но каждый вызов методов поиска будет сопровождаться выводом предупреждений.

### Либо class, либо полное имя

Экземпляр экшна/пака теперь можно запросить у ControllerCache используюя

- **класс** экшна/пака
- **полное имя** класса строкой

ВАЖНО: Поиск только по имени класса и поиск по ``short_name`` работать не будет!

#### Полное имя

Полное имя пака имеет вид ``'m3_ext.demo.actions.base.Pack'``, т.е. полный путь от корня проекта до класса пака, включая имя класса.

Полное имя экшна имеет подобный вид. Пример: ``'m3_ext.demo.actions.menus.MenuAction'``

Имя экшна/пака можно получить в консоли проекта (``manage.py shell``) примерно так:

    ::bash
    $ python manage.py shell
    ...
    >>> from project.app.actions import Pack
    <project.app.actions.Pack object at 0xa53882c>
    >>> Pack.get_short_name() # справедливо и для экшна
    'project.app.actions.Pack'
    >>> # Теперь можно искать
    >>> from m3.actions import ControllerCache as CC
    >>> CC.populate()
    True
    >>> СС.find_pack(Pack) is CC.find_pack('project.app.actions.Pack')
    True
    >>> CC.find_pack(Pack)
    <project.app.actions.Pack object at 0x......>

### find_action -> find_pack + action в атрибуте

Рекомендуется не искать экшн, а искать пак, его содержащий, а затем получать экшн из атрибута пака (разумеется, экшн нужно положить в атрибут пака в ``Pack.__init__``). Возможно, в дальнейшем поиск экшнов будет убран!


## <a name="overrides">Overrides</a>

В **m3-core** с версии **2.0.11** появился механизм "перегрузки" **ActionPack**'ов одних приложений в других приложениях. При этом поиск оригинального пака средствами **ControllerCache** будет возвращать экземпляр пака-заменителя.
Типичное применение - расширение поведения plugin'ами: plugin перегружает пак некоего реестра паком-потомком, при этом ярлык на рабочем столе будет вызывать экшн нового пака.

### Пример

#### Ядро проекта

**project/some_app/actions.py**

    ::python
    ...
    class SomePack(ActionPack):

        title = u'Реестр'
        win_action_cls = SomeAction

        def __init__(self):
            super(SomePack, self).__init__()
            self.win_action = win_action_cls()
    ...

**project/some_app/app_meta.py**

    ::python
    ...
    from m3_ext.ui.app_ui import DesktopLoader, DesktopShortcut
    from m3.actions import ControllerCache
    from actions import SomePack
    ...
    def register_actions():
        controller.packs.append(SomePack())
    ...
    def register_desktop_menu():
        # получаем экземпляр пака из контроллера
        pack = ControlleCache.find_pack('project.some_app.SomePack')

        # формируем Desktop-элемент для экшна полученного пака
        shortcut = DesktopShortcut(
            name=pack.title,
            pack=pack.win_action # шорткату пожно передавать экшн вместо пака
        )

        # добавляем на Рабочий Стол
        DesktopLoader.add(
            metarole=GENERIC_USER_METAROLE,
            place=DesktopLoader.DESKTOP,
            element=shortcut,
        )

#### Плагин

**project/some_plugin/actions.py**

    ::python
    ...
    from project.some_app.actions import SomePack
    ...
    class AdvancedPack(SomePack):
        # переопределяем класс окна
        win_action_cls = AdvancedAction
    ...

**project/some_plugin/app_meta.py**

    ::python
    ...
    from actions import AdvancedPack
    ...
    # ВАЖНО: Новый пак не нужно регистрировать в контроллере!
    # Перегрузка делается так:
    action_pack_overrides = {
        # полное имя пака            перегружающий экземпляр
        'project.some_app.SomePack': AdvancedPack()
    }


Т.о. AdvancedPack будет заменять оригинальный в интерфейсе, в т.ч. и в **ExtDictSelectField**'ах и гридах, коль скоро оные будут использовать экземпляр пака, получая его из ControllerCache.find_pack.
