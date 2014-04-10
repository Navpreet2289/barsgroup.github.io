<!-- 
.. title: Changelog 08.04.2014
.. slug: changelog-08-04-2014
.. date: 2014/04/08 11:03:52
.. tags: 
.. link: 
.. description: 
.. type: text
-->

** m3-core==2.0.11 **

- **BREAKING CHANGE:** Теперь нельзя регистрировать классы/строки в контроллеры - только экземпляры!
Уже починенные контрибы:
`m3-users>=2.0.1`, `m3_kladr>=2.0.2`. [Подробности здесь](/stories/comment-changelog-08-04-2014.html#find-packs-and-actions).
- Метод проверки прав _has\_sub\_permission_ заменен на _has\_perm_ _m3.actions.dicts.BaseTreeDictionaryActions_ и
_m3.actions.dict.BaseTreeDictionaryModelActions_
- Выпилен _list_view_ в _m3.actions.dicts.BaseTreeDictionaryActions_ и
_m3.actions.dict.BaseTreeDictionaryModelActions_
- Теперь доступен _override_ для одних паков другими. [Подробности здесь](/stories/comment-changelog-08-04-2014.html#overrides).
- _ActionController.append_pack_ теперь возвращает фактически добавленный пак
- Поправлен поиск экшнов/паков по правам. [Подробности здесь](/stories/comment-changelog-08-04-2014.html#find-packs-and-actions).
- В Django 1.6 удалили логгер по-умолчанию.
Оставляется обратная совместимость и поддержка джанги 1.6

**m3-ext==2.0.6**

- **BREAKING CHANGE:** Поиск паков/экшнов теперь должен осуществляться через _ControllerCache_!
- _DesktopModel_ изменена в пользу _view_ предоставяемой платформой.
Требуется адаптация проектных _view_. [Подробности здесь](/stories/comment-changelog-08-04-2014.html#desktop-view).
- Фильтрация элементов _desktop_ исходя из прав пользователя.
- Поправлены ошибки в _make\_read\_only_
- Поправлен _ExtObjectGrid.js_ касательно кроссбраузерной совместимости
- Поправлена ошибка в _object grid editor_
- Использование _absolute\_url_ заменено на _get\_absolute\_url_
- _ExtGridLockingHeaderGroupPlugin_ теперь может конфигурировать грид
- Реализован компонент _ObjectSelectionPanel_ с возможностью запоминания выбора значений в дополнительном гриде

**objectpack==2.0.21**

- В _TreeObjectPack_ поправлено сохранение корневых элементов
- Поправлено добавление на _desktop_ нескольких _submenu_ за один раз (кортежем)
- _observer_ теперь обрабатывает _overrided_ паки (m3-core>=2.0.11)
- Добавлены точки расширений для _ObjectEditWindowAction_