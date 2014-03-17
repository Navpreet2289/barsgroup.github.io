.. title: Release m3 1.0
.. slug: release-m3-10
.. date: 2013/01/01 16:54:07
.. tags: 
.. link: 
.. description: 
.. type: text

Набор изменений от 01-01-2013
=============================

В то далекое время весь код находился внутри одной библиотеки m3.

* В базовое окно для линейного, иерархического и сомещенного справочника ``m3.ui.ext.windows.complex.ExtDictionaryWindow``
  добавлен режим множественного выбора.
* В класс ``m3.ui.ext.panels.grids.ExtMultiGroupinGrid`` добавлен атрибут ``header_style`` -- стиль заголовка колонки.
  А также метод ``render_base_config``.
* В ``m3.ui.ext.misc.store.ExtDataStore`` добавлена возможность прописывать в шаблоне даты.
* В методе ``render_base_config`` класса ``m3.ui.ext.fields.simple.ExtStringField`` реализовано экранирование значений с
  обратным слэшем.
* В модуль ``m3.ui.ext.containers.grids`` добавлены два класса ``ExtLiveGridCheckBoxSelModel`` и ``ExtLiveGridRowSelModel``
* В класс ``m3.ui.ext.containers.forms.ExtForm`` добавлен метод ``try_to_list``.
* В класс ``m3.ui.ext.containers.forms.ExtPanel`` добавлен атрибут ``title_collapse``. Атрибут указывает нужно ли сворачивать
  панель при щелчке на заголовке.
* В класс ``m3.ui.actions.dicts.tree.BaseTreeDictionaryActions`` добавлен метод ``get_default_action``, который возвращает
  экшн по-умолчанию.
* В класс ``m3.ui.actions.dicts.tree.BaseTreeDictionaryModelActions`` добавлена настройка ``list_drag_and_drop`` разрешающая
  перетаскивание элементов из грида в другие группы дерева.
* В класс ``m3.ui.actions.dicts.simple.BaseDictionaryActions`` добавлен метод ``get_default_action``, который возвращает
  экшн по-умолчанию.
* В класс ``m3.ui.actions.results.ActionRedirectResult`` добавлен атрибут ``context`` и метод ``prepare_request``.
* В метод ``convert_value`` класса ``m3.ui.actions.context.ActionContext`` добавлена проверка версии Python, т.к. в версиях
  ниже 2.7 в `Decimal` не поддерживается создание из `float`.
* В классе ``Action`` добавлена проверка прав в родительском элементе из экшена
* В класс ``m3.ui.app_ui.DesktopLauncher`` добавлен метод ``_set_default_handler``
* Добавлен модуль ``m3_tags``. Там описан темплейт таг, который возвращает URL экшена.
* В метод ``deleteOkHandler`` классa ``Ext.m3.ObjectGrid`` добавлена проверка на ошибки уровня приложения.
* В класс ``Ext.m3.MultiSelectField`` добавлен метод ``fireChangeEventOnDemand``, который имитирует поведение ``Ext.form.Field.onBlur()``.
* В класс ``Ext.ux.form.FileUploadField`` добавлены события ``change`` (отрабатывает, когда изменилось значение) и
  ``beforechange`` (срабатывает до изменения поля)
* В класс ``Ext.m3.EditWindow`` добавлен метод ``clearModificationFlag``, который отбрасывает признаки модифицированности формы.
* Для ``Ext.m3.AdvancedComboBox`` корректирована функция ``getWidth``, которая вычисляет ширину поля.
* В класс ``Ext.m3.AddrField`` добавлено событие ``change_corps``, которое срабатывает при изменение корпуса.
* В класс ``m3.helpers.datagrouping.RecordProxy`` добавлен атрибут ``grouped`` -- список атрибуто, по которым происходит
  группироровка
* В класс ``m3.helpers.datagrouping.GroupingRecordProvider`` добавлен атрибут ``detail_attrs_map`` -- словарь, определяющий
  дополнительные поля, которые добавляются в узловые записи, создаваемые самим провайдером.
* Добавлен контриб `setupenv`
* Добавлен контриб `prepare_env`
* Множественные исправления, которые можно увидеть выполнив, например, визуальное сравнение веток через TortoiseHG.