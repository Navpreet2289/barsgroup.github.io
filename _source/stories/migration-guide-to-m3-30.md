<!-- 
.. title: Инструкция по переходу на M3 3
.. slug: migration-guide-to-m3-30
.. date: 2014/08/18 15:56:36
.. tags: 
.. link: 
.. description: 
.. type: text
-->

## Содержание

- [Ключевые отличия от версии M3 2](#key-difference)
- [Преимущества](#advantage)
- [Перевод продукта](#product-refactoring)
    - [Что нужно сделать вначале?](#starting-guide)
    - [Перевод экшенов](#actions-refactoring)
        - [Стандартные экшены и паки](#standart-actions-packs)
        - [Что стало с биндингом данных?](#binding)
        - [Сериализация объекта модели](#serialization)
        - [Изменения в описании Context Declaration](#context-declaration)
        - [Перевод справочников на objectpack](#migration-to-objectpack)
        - [Компоненты master-detail](#master-detail)
    - [UI](#ui)
        - [UI на сервере](#server-ui)
        - [UI на клиенте](#client-ui)
        - [Концептуальные отличия](#ui-key-difference)
        - [Promises](#promises)
        - [BREAKING CHANGES конструкций UI](#ui-breaking-changes)
    - [Пример перевода](#example)
        - [Пример написания кода на версии m3 2 и ранее](#example-in-2)
        - [Как решается эта же задача на версии m3 3](#example-in-3)


## <a name="key-difference">Ключевые отличия</a>

- Браузер взаимодействует с сервером исключительно через json;
- Отказ от django templates (template-globals) в пользу static js-файлов;
- Если в m3 2 для получения формы с данными используется один ajax-запрос на сервер.
То в версии 3 будет последовательно генерироваться три запроса:

    - Запрос за данными;
    - Запрос за ui (json-конфигурация);
    - Запрос за статической js-частью (js-логика).

- UI формируется по прежнему на сервере через python;
- Полная поддержка классов Action, ActionPack;
- "Справочниковые" экшены поддерживаются только
[ObjectPack-ом](http://objectpack.docs.bars-open.ru/).
Поэтому классы наследоваемые от паков или экшенов

    - m3.actions.dics.*
    - m3.actions.packs.*

необходимо перевести на objectpack.

## <a name="advantage">Преимущества</a>

- **Отделение представления от данных**. Существенно облегчит любой рефакторинг как уровня бизнес-логики, так и
уровня UI
- Большой шаг в сторону **перехода на [Ext JS 5.0](http://docs-origin.sencha.com/extjs/5.0/)**, за счет того, что js-код
стал 100% декларативен и описывается с помощью конструкций, совместимых с ExtJs 5.0 (таких как *Ext.define*,
*Ext.override*; атрибутов *extend*, *xtype*; использование при вызове род. метода - *callParent* )
- **Кеширование ui-конфигурации**. Так как зависимость между данными и интерфейсом пропала, в версии 3.0 js-файлы
нативно кешируются браузером и запрашиваются посредством *requirejs* однажды. В перспективе планируется сделать
прозрачное кеширование json-ответов, или даже сборку всех окон в статический json-файл, который предлагается
подключать на старте проекта. За счет чего нагрузка на сервер существенно снизится, так как сервер будет работать только
с данными.
- Серьезный шаг в сторону **RESTful-интерфейсов**, который позволит еще сильнее декомпозировать серверные механизмы и сделать
их менее монолитными и зависимыми от одного типа UI.
- Перспективная возможность использования **дизайнера для генерации UI-интерфейсов**, так как в качестве результата можно использовать
json-конфигурацию, а не python-код
- Увеличение скорости работы за счет **отказа от механизм django-шаблонов** в пользу static-файлов, которые отдает nginx
- Теоретическая возможность **тестирования по отдельности** клиента (UI) и сервера (данные)

## <a name="product-refactoring">Перевод продукта</a>

### <a name="starting-guide">Что нужно сделать вначале?</a>

Подготовка к запуску продукта на новой версии:

- Необходимо сделать минимальное рабочее приложение на текущей ветке (dev), которое бы
запускалось и могло бы отображать рабочий стол;
- Необходимо установить два виртуальных окружения: одно (*старое*) для рабочей версии
(будет являться примером), второе (*новое*) для версии, где будут происходить исправления;
- В рабочей версии нужно временно закоментировать максимальное количество приложений (модулей), без которых
система сможет запускаться, то же самое проделать на версии с обновленной платформой;
- В корне продукта нужно создать файл TRASH.py, где будут находиться заглушки удаленных компонентов.
Пример файла для продукта Род. плата можно посмотреть
[здесь](https://gist.github.com/prefer/de542c929413195f466c);
- Необходимо произвести переименование устаревших импортов на импорты из модуля TRASH.py
- Любые изменения, комментирования исходников сопровождаются тегом #NR,
для удобного последующего поиска;
- Затем в несколько этапов нужно включать приложения в INSTALLED_APPS
добавляя в TRASH.py новые классы-заглушки:

    - cначала зависимые модули;
    - затем оставшиеся приложения;
    - и, наконец, плагины и сопутствующий функционал.

- По мере включения что-то может ломаться, в таком случае необходимо дополнять файл TRASH.py
новыми классами-заглушками и делать импорты на этот файл;
- В итоге должны получить рабочий стол с настроенным меню “Пуск” и верхним topbar-ом.
Все окна остаются нерабочими.

## <a name="actions-refactoring">Перевод экшенов и паков</a>

### <a name="standart-actions-packs">Стандартные экшены и паки</a>

Главным отличием является то, что новой версией поддерживаются
исключительно стандартные классы:
[Action](http://m3-core.docs.bars-open.ru/ru/latest/contents.html?highlight=action#m3.actions.__init__.Action) и
[ActionPack](http://m3-core.docs.bars-open.ru/ru/latest/contents.html?highlight=action#m3.actions.__init__.ActionPack).
Для справочников и master-detail контейнеров необходимо использовать
[objectpack](http://objectpack.docs.bars-open.ru/).

Для отдачи UI появился новый базовый класс
[UIAction](https://bitbucket.org/barsgroup/m3-ext/src/b4d06a2cdce7a250c79100daab036606452f20de/src/m3_ext/__init__.py?at=client-rendering#cl-14)
Который включает в себя два метода:

- *get_ui* - возвращает экземпляр ExtUIComponent, либо словарь вида {
            "config" :: dict - базовый конфиг окна
            "data"   :: dict - базовые данные для инициализации окна
        }
- *get_result* - возвращает словарь вида {
            "ui":     :: str  - url для получения базового конфига окна
            "model"   :: dict - объект данных
            "data":   :: dict - данные для конкретного окна
        }

В версии m3 3 отправляются три запроса при открытии нового окна (в порядке отправки):

- Получение данных в формате json;
- Получение ui в формате json;
- Получение js-представления в формате javascript-файла;

Последовательность запросов при получении ui:

- Отправка запроса на *url*-экшена;
- Запрос приходит на сервер в метод *get_result* вместе с контекстом, возвращаются данные для формы;
- Клиент получив ответ с данными инициизирует запрос за ui по *url*-экшена с параметров *mode=ui*;
- Запрос приходит на сервер в метод *get_ui*, который возвращает экземпляр окна (или *ExtUIComponent*);
- Внутри m3-ext с помощью;
 [UIJsonEncoder](https://bitbucket.org/barsgroup/m3-ext/src/b4d06a2cdce7a250c79100daab036606452f20de/src/m3_ext/ui/results.py?at=client-rendering#cl-29)
 окно серриализуется в json-представление;
 - Клиент, получив ответ от сервера, в виде *json*-представления может сгенерировать запрос на *js*-представление по *xtype*
 компоненту, если такой *xtype* еще не зарегестрирован;
 - *js*-представление, находящиеся в статике, отдается веб-сервером.

Таким образом необходимо во всех экшенах, которые отдают ui, изменить класс-наследник от
UIAction. Экшены, не отдающие ui менять не нужно.

### <a name="binding">Что стало с биндингом данных?</a>

Под биндингом данных понимается автоматическое преобразование django-модели в extjs-форму и обратно. При условии, если
названия атрибутов модели и формы совподают.

Как было в версии m3 2 и раньше:

- django model -> extjs form (при загрузки имеющихся данных, например, при редактировании)

    В методе формы (ExtForm) был метод from_object, принимающий в качестве параметров объект модели,
     и производящий сопоставллениеполя формы с полями модели. Затем заполненное данными окно рендерилось через
     вызов метода render у окна. Таким образом получался js-код в совокупности с данными, который eval-лился на клиенте
     в браузере.

- extjs form -> django model (при сабмите данных, например, при сохранении элемента справочника)

    В методе формы (ExtForm) был метод bind_to_request, принимающий параметр - request. Данный метод производил
    заполнение формы.

    В методе формы (ExtForm) был метод to_object, принимающий в качестве параметра объект модели,
    который необходимо заполнить данными из окна. Затем модель валидировалась и сохронялась.

Как стало в версии m3 3:

- django model -> extjs form

    При использовании ExtEditWindow достаточно соблюсти правило биндинга - название атрибутов модели и
    UI-окна должны совпадать. Если по неким причинам название не совподают, то можно в js-представлении переопределить
    метод bind и внутри метода установить необходимый параметр или зависимость параметров.

- extjs form -> django model

    Не изменился.

### <a name="serialization">Сериализация объекта модели</a>

При использовании objectpack появилась возможность указывать правила сериализации из объекта в словарь. Для этого
необходимо добавить метод serialize(include=None, exclude=None) внутри которого объект модели доступен через self,
необходимо вернуть словарь. Внутри можно использовать функцию [model_to_dict](https://bitbucket.org/barsgroup/m3-core/src/8ba2e89984028d2584d94acc5b2f17660199ab3f/src/m3/db/tools.py?at=client-render#cl-51)
для сериализации модели.

Так же крайне желательно проводить сериализацию внутри модели и тогда, когда objectpack не используется. В этом случае
внутри метода get_result экшена необходимо у полученного объекта вызвать метод serialize. Например:

    ::python

    class RemaindersDocDetail(BasePaidServObjectModel):
        """
        Табличная часть документа остатков
        """
        remaindersdoc = models.ForeignKey(RemaindersDoc)
        consumer = models.ForeignKey("domain.Consumer")
        servicepoint = models.ForeignKey("domain.ServicePoint")
        service = models.ForeignKey("domain.Service", verbose_name=u'Услуга', blank=True, null=True)
        bank_props = models.ForeignKey('domain.BankProps', verbose_name=u'Платежные реквизиты', blank=True, null=True)
        type = models.SmallIntegerField(choices=RemaindersTypeEnum.get_choices(),
                                        null=False, default=RemaindersTypeEnum.DEBT)
        summa = models.DecimalField(null=False, blank=False, max_digits=16, decimal_places=2)
        out = models.BooleanField(verbose_name=u'Признак выбывшего', default=False)

        class Meta:
            db_table = 'finances_remainders_details'
            verbose_name = u"Табличная часть остатков"

        def serialize(self, *args):
            return model_to_dict(self)


    class RemaindersEditWindowAction(UIAction):
        """
        Окно редактирования остатков
        """
        url = '/edit-window'

        def context_declaration(self):
            return [ACD(name="remainders_detail_id", type=int, required=True, default=0),
                    ACD(name="consumer_id", type=int, required=True, default=-1),
                    ACD(name='sp_id', type=int, required=True, default=-1),
                    ACD(name='period', type=int, required=True),
                    ACD(name="global_provider_id", type=int, required=True)]

        def get_result(self, request, context):
            result = super(RemaindersEditWindowAction, self).get_result(request, context)
            if context.remainders_detail_id:
                detail = RemaindersDocDetail.objects.get(pk=context.remainders_detail_id)
            else:
                detail = RemaindersDocDetail()

            read_only = False
            if context.remainders_detail_id and detail.remaindersdoc.state == RemaindersDocStateEnum.REGISTERED:
                read_only = True

            if not urls.get_pack_instance('service-facts').has_sub_permission(
                    request.user,
                    urls.get_pack_instance('service-facts').REMAINDERS_PERMISSION,
                    request
            ):
                read_only = True

            result['model'] = detail.serialize()
            result['data'].update({
                'read_only': read_only,
                'submit_url': self.parent.save_action.absolute_url()
            })
            return result

        def get_ui(self, request, context):
            return RemainderDetailEditWindow()


### <a name="context-declaration">Изменения в описании Context Declaration</a>

Важное замечание: в ui-экшенах декларация контекста производится так же как и раньше - через определение метода
context_declaration, но он декларирует параметры исключительно для метода get_result. Так как метод get_ui ничего не
должен знать о контексте, поэтому туда контекст и не приходит.

В версии 3 контекст может описываться с помощью словаря, пример:

    ::python

    def context_declaration(self):
        return {
            'servicepoint_id': {'type': 'int'},
            'assigned_service_id': {'type': 'int_or_none',
                                    'verbose_name': u"Идентификатор типовой услуги"
            }
        }

Это более предпочтительный пример, чем:

    ::python

    def context_declaration(self):
        return [
            ACD(name='start', type=int, required=True, default=0),
            ACD(name='limit', type=int, required=True, default=25),
            ACD(name='date_since', type=date, required=False),
            ACD(name='date_until', type=date, required=False),
            ACD(name='filter', type=str, required=False),
            ACD(name='servicepoint_id', type=int, required=True, default=0),
        ]

Так как не нужно импортировать и использовать дополнительные классы, такие как [ACD](http://m3-core.docs.bars-open.ru/ru/latest/simple_usage.html?highlight=acd#m3.actions.context.ActionContextDeclaration).
К тому же удален параметр *required*,
поэтому если указан параметр *default* - то параметр будет считаться не обязательным. *ACD* объявлен как **deprecated**.

### <a name="migration-to-objectpack)">Перевод справочников на objectpack</a>

С версии M3 3 поддерживается исключительно механизм работы через [objectpack](http://objectpack.docs.bars-open.ru/)
со справочниками или с сущностями, которые
поддерживают операции создания/редактирования/удаления. То есть считаются устаревшими механизмы работы через классы
*BaseDictionaryActions*, *BaseDictionaryModelActions*, *BaseTreeDictionaryActions*, *BaseTreeDictionaryModelActions*.

Поэтому перед переводом на версию M3 3 необходимо в качестве первой итерации перевести все подобные механизмы на
objectpack.


### <a name="master-detail">Компоненты master-detail</a>

Comming soon...

## <a name="ui">UI</a>

### <a name="server-ui">UI на сервере</a>

В основном изменения затронули механизм рендеринга UI.

Основные отличия:

- отказ от django-шаблонов в пользу сериализации в json-представлени;
- хранение внутри \__slots\__;
- отказ на клиенте от eval в пользу Ext.create

Ключевое отличие - это отказ от django-шаблонизатора
в пользу сериализация в json-представление. Фактически с версии m3 3 ui-представление должно возвращаться с сервера в виде
json, которое в последствии на клиенте сериализуется и передается как параметр в функции *Ext.create*.
Таким образом не используется *eval*, что существенно облегчает отладку.
UI можно описывать как декларативно - через python-словарь, так и в старом стиле через описание классов.

Пример использования декларативного стиля:

    ::python

    win = {
        'xtype': 'm3-window',
        'height': 500,
        'width': 400,
        'items': [
            'xtype': 'form',
        ]
    }

Аналогично:

    ::python

    win = ExtWindow(
        height=500,
        width=400,
        items=[
            ExtForm()
        ]
    )

Или:

    ::python

    win = ExtWindow()
    win.height = 500
    win.width = 400
    win.items.append(ExtForm())


### <a name="client-ui">UI на клиенте</a>

Более подробную информацию по каждому компоненту и его работе можете найти в
[примерах использования](https://bitbucket.org/barsgroup/m3-ext/src/1cac8604bcc3c4e2979d014a062b60fa5c91e960/src/m3_ext/demo/?at=client-rendering).
Достаточно поставить проект [m3-blank](https://bitbucket.org/barsgroup/m3-blank) и подключить в качестве приложения
*m3_ext.demo*

До версии m3 3 функцию генерации UI выполнял django-шаблонизатор, который на каждый запрос отдавал js-представление.
Оно впоследствии eval-лилось на клиенте с помощью функции *smart_eval*.

Это выглядело примерно так:

    ::javascript

    var form = Ext.getCmp("{{component.form.client_id}}");
    var tab_panel = Ext.getCmp("{{component.tab_panel.client_id}}");
    var other_props = Ext.getCmp("{{component.other_props.client_id}}");
    var inn2 = Ext.getCmp("{{component.inn2.client_id}}");
    var kpp2 = Ext.getCmp("{{component.kpp2.client_id}}");
    var name_other_org = Ext.getCmp("{{component.name_other_org.client_id}}");

    win.on('beforesubmit',function(){
        for (i=1;i<tab_panel.items.length;i++){
            tab_panel.activate(i);
        }
    });

    function fill_bankprop_fields(args){
        Ext.getCmp("{{component.bank1.client_id}}").setValue(args.namep);
        Ext.getCmp("{{component.bik.client_id}}").setValue(args.newnum);
        Ext.getCmp("{{component.bank_filial.client_id}}").setValue('');
        Ext.getCmp("{{component.correspondent_account.client_id}}").setValue(args.ksnp);
        Ext.getCmp("{{component.short_address.client_id}}").setValue(args.nnp);
    }

    function selectBankTemplate(){
        if (form) {
            var baseParams = {};
            Ext.Ajax.request({
                url: '{{ component.select_bank_url }}',
                params: Ext.applyIf(baseParams || {}, win.actionContextJson || {}),
                success: function(response, opts){
                    smart_eval(response.responseText);
                },
                failure: uiAjaxFailMessage
            });
        }
        return false;
    }

    other_props.on("check" ,setReadOnlyFunc);
    setReadOnlyFunc(other_props);

    function setReadOnlyFunc(opt){
        if(opt.getValue()){
            inn2.setReadOnly(false);
            kpp2.setReadOnly(false);
            name_other_org.setReadOnly(false);
        }else{
            inn2.setReadOnly(true);
            kpp2.setReadOnly(true);
            name_other_org.setReadOnly(true);
        }
    }

    win.on("fill_fields", function(args){
        //проверяем перед заменой, заполнены ли поля
        var not_empty_str = Ext.getCmp("{{component.bank1.client_id}}").value +
                            Ext.getCmp("{{component.bik.client_id}}").value +
                            Ext.getCmp("{{component.bank_filial.client_id}}").value +
                            Ext.getCmp("{{component.correspondent_account.client_id}}").value +
                            Ext.getCmp("{{component.short_address.client_id}}").value;
        if(not_empty_str.length > 0) {
            Ext.Msg.show({
                title: 'Перезапись данных',
                msg: 'Некоторые поля банковских реквизитов уже заполнены.<br />Произвести перезапись?',
                buttons: Ext.Msg.YESNO,
                fn: function(btn, text){
                    if (btn == 'yes'){
                        fill_bankprop_fields(args);
                    }
                },
                icon: Ext.Msg.WARNING
            });
        } else {
            fill_bankprop_fields(args);
        }
    });

- *component.form.client_id* - Возвращается идентификатор, по которому в дальнейшем производится поиск компонента
в отрендеренном пространстве браузера;
- *component.select_bank_url* - так передается url для последующего Ajax-запроса.

Как стало сейчас:

    ::javascript

    Ext.define('Ext.paidserv.BankPropsDictAddWindow', {

        extend: 'Ext.m3.EditWindow',
        xtype: 'bank-props-dict-add-window',

        initComponent: function () {
            this.callParent();

            this.tab_panel = this.findByItemId('tab_panel');
            this.other_props = this.findByItemId('other_props');
            this.inn2 = this.findByItemId('inn2');
            this.kpp2 = this.findByItemId('kpp2');
            this.name_other_org = this.findByItemId('name_other_org');

            this.bank1 = this.findByItemId('bank1');
            this.bik = this.findByItemId('bik');
            this.bank_filial = this.findByItemId('bank_filial');
            this.correspondent_account = this.findByItemId('correspondent_account');
            this.short_address = this.findByItemId('short_address');

            this.on('beforesubmit', function () {
                var i = 0;
                for (i = 1; i < this.tab_panel.items.length; i++) {
                    this.tab_panel.activate(i);
                }
            }, this);

            this.other_props.on("check", this.setReadOnlyFunc, this);
            this.on('afterrender', this.setReadOnlyFunc, this);
            this.bank1.on('beforerequest', function(cmp, req){
                if (req) {
                    req.success = function (win) {
                        win.on('select', this.onSelectBank, this);
                        return win;
                    }.bind(this)
                }
            }, this);
            this.bank1.on('change', this.onChangeBank, this);
        },

        onSelectBank: function(cmp, id, displayText){
            // достанем запись о банке
            var bank_rec = cmp.grid.getSelectionModel().getSelected();
            this.bank1.setRecord(bank_rec);
        },

        onChangeBank: function(){
            var bank_rec = this.bank1.getRecord();
            if (bank_rec) {
                this.bik.setValue(bank_rec.json['newnum']);
                this.bank_filial.setValue('');
                this.correspondent_account.setValue(bank_rec.json['ksnp']);
                this.short_address.setValue(bank_rec.json['nnp']);
            }
        },

        setReadOnlyFunc: function () {
            if (this.other_props.getValue()) {
                this.inn2.setReadOnly(false);
                this.kpp2.setReadOnly(false);
                this.name_other_org.setReadOnly(false);
            } else {
                this.inn2.setReadOnly(true);
                this.kpp2.setReadOnly(true);
                this.name_other_org.setReadOnly(true);
            }
        },

        bind: function(data){
            this.callParent(arguments);
        }

    });

### <a name="ui-key-difference">Концептуальные отличия</a>

- Каждый ui компонент на сервере должен иметь определенный **xtype**, каждый extjs-компонент должен зарегистрировать xtype
- По аналогии с *xtype*, каждый плагин должен зарегестрировать **ptype**. Пример использования *ptype* в python-коде:

        ::python

        # Было
        self.charge_grid.plugins.append('new Ext.ux.grid.GridSummary()')

        # Стало
        self.charge_grid.plugins.append({'ptype': 'gridsummary'})

- js-представление, должно находиться в **static**-файлах так же должно иметь *xtype*. Для этого в *settings.py* в кортеж
**STATICFILES_FINDERS** должен быть добавлен следующий элемент - *'m3.finders.RecursiveAppDirectoriesFinder'* -
это позволит иметь папку *static* в любой вложенности приложения
- Фактически на каждый старый *template-globals* необходимо создать новый класс в стиле ExtJs
- Получение ссылки на вложенный компонент производится через вызов метода окна - **findByItemId**, поиск ведется по
атрибуту *itemId*. Этот параметр можно задавать из python-кода, по-умолчанию подставляется название атрибута, например:

        ::python

        class MyWindow(ExtWindow):

            _xtype = 'my-window'

            def __init__(self):
                self.form = ExtForm()
                self.items.append(self.form)

    Такая конструкция равносильна python-коду

        ::python

        class MyWindow(ExtWindow):

            _xtype = 'my-window'

            def __init__(self):
                self.items.append(ExtForm(item_id='form'))


    Это эквивалентно json-представлению вида:

        ::javascript

        {
            "xtype": "my-window",
            "items": [
                "xtype": "form",
                "itemId": "form"
            ]
        }

- Все функции превратились в **методы** класса *Ext.paidserv.BankPropsDictAddWindow*
- Инициализация ссылок на компоненты производится в методе **initComponent**
- В методе **bind** в качестве параметра *data* могут прийти различные urls, данные для формы и прочие данные, которые
необходимо обработать, если они не обрабатываются в классах-наследниках.
- Скорость поиска компонент заметно улучшилась, так как глобальная функция *Ext.getCmp* работает объективно медленне
поиска внутри компонента по *itemId*
- Нет надобности использовать уникальные идентификаторы *client_id* для компонентов
- Удалена функция *smart_eval*, так как нет необходимости eval-ить полученый javascript-код, так как вместо него возвращается
json

### <a name="promises">Promises</a>

С версии m3 3 в javascript появилась возможность использовать механизм обещаний из библиотеки [q.js](https://github.com/kriskowal/q).
Примеры работы и документацию можно найти на [офф. сайте](http://documentup.com/kriskowal/q/#tutorial).

#### <a name="">API на основе promise-ов</a>

- **UI.evalResult** - Сериализует в json и обрабатывает полученное значение, создавая экземпляр компонента
- **UI.ajax** - Загружает JSON AJAX-запросом и кладёт в promise, пример использования:

        ::javascript

        // Пример отправки запроса на сервер и отображение окна, если в качестве результата возвращается окно
        UI.ajax({
            url: data.url,
            params: data.context
        }).then(UI.evalResult)
            .catch(uiAjaxFailMessage);

- **UI.callAction** - Производит вызов ajax-запроса, обработку его и возвращает promise. Внутри себя производит все
необходимые действия по отображению окна, работе с масками, модальностью. Примеры:

        ::javascript

        // Пример отправки запроса на сервер с параметрами, получения json-результата и обработка результата
        UI.callAction.call(this, {
            url: this.loadSelectedQueryUrl,
            method: 'POST',
            params: {
                query_id: selected_query
            },
            success: function(obj){
                this.query_str.setValue(obj['query']);
            }.createDelegate(this),
            failure: uiAjaxFailMessage
        });

    В случае успеха будет установлено значение в поле *this.query_str*.

        ::javascript

        // Пример отправки запроса на сервер, получение json-конфигурации окна и подписка на событие
        UI.callAction.call(this, {
            url: this.grid_edit_url,
            params: params,
            success: function (win) {
                win.on('addRow', function (data) {
                    this.saveRec(data);
                }, this);
            }.bind(this)
        });

    В случае успеха происходит подписание экземпляра окна на событие *addRow* совместно с реализацией обработчика.

- **UI.require** - подгрузка модулей и их зависимостей. Пример использования:

        ::javascript

        // Пример отправки запроса на сервер, сириализация ответа и подтягивание зависимостей
        // перед созданием и отображением окна
        UI.ajax({
            method: 'POST',
            url: this.rule_edit_url,
            params: Ext.apply(baseParams, this.getContext() || {})
        }).then(UI.evalResult)
        .then(function(result) {
            if (result.config) {
                // Подтягиваем зависимости
                UI.require([result.config['xtype']])
                .spread(function () {
                    return [result.config, result.data];
                }).spread(UI.createWindow);
            }
        })
        .catch(uiAjaxFailMessage);

### <a name="ui-breaking-changes">BREAKING CHANGES конструкций UI</a>

- В классе *ExtGridColumn* атрибут **column_render** переименовался в атрибут **render**
- В классах *ExtGrid* и *BaseExtTriggerField* (соотвественно, *ExtComboBox* и *ExtDictSelectField*) перестали существовать
методы **set_store** и **get_store**. Так сейчас имеет место быть декларатиное описание компонентов, устанавливать *store*
необходимо через прямое присваивание.
- Класс **ExtTreeNode** перестал существовать, вместо него необходимо использовать стандарный **dict**
- Переименован атрибут *paging_bar* в компоненте *ExtGrid* в **allow_paging**
- Атрибут класса *ExtGridColumn* **summaryType** раньше хранил значения вида, пример:

        ::python

         {
            'header': u'Сумма',
            'data_index': 'summa',
            'sortable': True,
            'align': 'right',
            'renderer': 'FloatRenderer',
            'width': 20,
            'extra': {'summaryType': '"sum"'},
        },

    **sum**, сейчас необходимость использовать "строку в строке" пропала, нужно использовать **sum** как значение,
    например:

        ::python

         {
            'header': u'Сумма',
            'data_index': 'summa',
            'sortable': True,
            'align': 'right',
            'renderer': 'FloatRenderer',
            'width': 20,
            'extra': {'summaryType': 'sum'},
        },

- Все функции-хенлдеры или рендереры будут искаться в skope окна как обработчики, то есть при написании конструкции

        ::python

        self.grid = ExtObjectGrid(
            region='center',
            sm=ExtGridCheckBoxSelModel())
        self.grid.add_column(
            header=u'№',
            width=10,
            renderer="AutoRenderer")
        self.grid.add_column(
            header=u'Состояние',
            data_index='state_verbose',
            width=40,
            sortable=True)
        self.grid.add_column(
            header=u'Дата документа',
            data_index='docdate',
            width=40,
            sortable=True)
        self.grid.add_column(
            header=u'Дата операции',
            data_index='operdate',
            width=40, sortable=True)
        self.grid.add_column(
            header=u'Номер',
            data_index='docnum',
            width=21,
            sortable=True)
        self.grid.add_column(
            header=u'Позиции',
            data_index='count',
            width=24,
            sortable=True)
        self.grid.add_column(
            header=u'Сумма',
            data_index='summa',
            width=25,
            align="right",
            sortable=True,
            renderer="FloatRenderer")
        self.grid.add_column(
            header=u'Тип',
            data_index='type_verbose',
            width=60,
            sortable=True)
        self.grid.add_column(
            header=u'Комментарий',
            data_index='comment',
            width=110,
            sortable=True)
        # кнопки в верхней панели грида
        self.grid.top_bar.items.append(ExtButton(
            text=u'Зарегистрировать',
            icon_cls='x-form-file-icon',
            handler='enableRegistration'))

    Необходимо, чтобы функция *AutoRenderer*, *FloatRenderer* и *enableRegistration* были в ExtJs-классе окна, например:

        ::javascript

        Ext.define('Ext.paidserv.CorrDocsArchiveWindow', {
            extend: 'Ext.m3.Window',
            xtype: 'corrdocs-archive-window',

            AutoRenderer: AutoRenderer, // ссылка на глобальную функцию
            FloatRenderer: FloatRenderer,
            StateRenderer: StateRenderer,



            bind: function (data) {
                this.callParent(arguments);
                if (data.readOnly) {
                    this.setBlocked(data.readOnly, data.readOnlyExclude);
                    this.grid.getTopToolbar().getComponent('button_edit').setText('Просмотр');
                }
                this.setTitle(this.title + data.title);
                this.date_since.setValue(data.dateSince);
                this.date_until.setValue(data.dateUntil);
                this.register_submit_url = data.register_submit_url;

                this.debtWindowUrl = data['debtWindowUrl'];
            },

            ...

            enableRegistration: function () {
                var actionType = true;
                this.registerAction(actionType);
            },

            registerAction: function (actionType) {
                //получаем выделенные строки
                var selRecords = this.grid.getSelectionModel().getSelections(),
                    selectedId = [],
                    i,
                    maskText = actionType == true ? 'Регистрация документа' : 'Снятие регистрации документа';

                for (i = 0; i < selRecords.length; i++) {
                    selectedId[i] = selRecords[i].id;
                }

                UI.callAction.call(this, {
                    loadMaskText: maskText,
                    url: this.register_submit_url,
                    params: Ext.applyIf({'ids': selectedId.join(','),
                        "action_type": actionType}, this.getContext()),
                    success: function (response, opts) {
                        this.grid.getSelectionModel().clearSelections();
                        this.grid.store.load();
                    }.bind(this),
                    failure: uiAjaxFailMessage
                });
            }
        });

- Однако функция **summaryRenderer** не может быть задана в качестве строкового представления в python-коде, такую функцию
необходимо устанавливать в extjs-представлении, например:

        ::javascript


            setSummaryRenderer: function (grid, name, func) {
                var cm, col;
                cm = grid.getColumnModel();
                col = cm.findColumnIndex(name);
                cm.config[col].summaryRenderer = func;
            }

            setSummaryRenderer(this.grid, 'state_verbose', function () {
                return 'Итого по документам:'
            });

- Все вызовы *Ext.Ajax.request*, которые внутри себя вызывали **smart_eval** должны быть переписаны в концепции с
функции *UI.callAction*
- Удалены классы **ExtJsonReader** и **ExtDataReader**, вместо использования этих классов необходимо все параметры
передавать в **ExtStore**, например:

        ::python

        # Было
        self.reader = ExtJsonReader(total_property='total', root='rows')
        self.reader.set_fields(
            'id',
            'identical',
            'tab_num',
            'fio', 'consumer_id',
            'service_ref_name', 'service_id',
            'bank_props_name', 'bank_props_id',
            'servicepoint_ref_name', 'servicepoint_id',
            'summa')

        self.grid.store = ExtGroupingStore(auto_load=True, root='rows', id_property='id')
        self.grid.store.reader = self.reader

        # Стало
        self.grid.store = ExtGroupingStore(
            auto_load=True,
            root='rows',
            id_property='id',
            total_property='total',
            root='rows',
            fields=[
                'id',
                'identical',
                'tab_num',
                'fio', 'consumer_id',
                'service_ref_name', 'service_id',
                'bank_props_name', 'bank_props_id',
                'servicepoint_ref_name', 'servicepoint_id',
                'summa']
            )

- Атрибут **triiger_action_all** в класс *ExtDictSelectField* был переименован в **trigger_action**, пример использования:

        ::python

        # Было
        self.servicepoint_field = fields.ExtDictSelectField(
            anchor='90%',
            name='servicepoint',
            display_field='name',
            value_field="id",
            label=u'Группа',
            trigger_action_all=False, # False = query, True = all
            ask_before_deleting=False,
            hide_trigger=False,
            hide_edit_trigger=True,
            width=190)

        # Стало
         self.servicepoint_field = fields.ExtDictSelectField(
            anchor='90%',
            name='servicepoint',
            display_field='name',
            value_field="id",
            label=u'Группа',
            trigger_action=ExtDictSelectField.QUERY,
            ask_before_deleting=False,
            hide_trigger=False,
            hide_edit_trigger=True,
            width=190)

- Было переименовано событие **closed_ok** в **select**, когда происходит выбор элемента в окне справочника.

## <a name="example">Пример перевода</a>

Возьмем для примера задачу, которая отрисовывает окно. Причем окно наследуется от продуктового базового класса, которое
имеет определенную логику на javascript.

### <a name="example-in-2">Пример написания кода на версии m3 2 и ранее</a>

Реализация экшена:

    ::python

    class CompensationInfoReportWindowAction(Action):
        """
        Вызов диалога "Информация о компенсации"
        """
        shortname = "compensation-info-report-window"
        url = '/%s' % shortname
        verbose_name = u'Окно настроек печати'

        def run(self, request, context):
            win = ui.CompensationInfoReportWindow(self)
            return ExtUIScriptResult(win)

Реализация UI:

    ::python

    class CompensationInfoReportWindow(BaseReportWindow):
        """
        Окно настроек печати оборотки по районам
        """
        def __init__(self, parent=None):
            super(CompensationInfoReportWindow, self).__init__()
            self.form.url = urls.get_url("compensation-info-report")
            self.title = parent.verbose_name if parent else u''

            # скорректируем высоту
            self.height = 370
            self.sp_grid.height = 290

            # соберем нужные блоки
            self.form.items.extend([
                self.sp_cont
            ])

Класс-наследник для UI *BaseReportWindow*:

    ::python

    class BaseReportWindow(ExtWindow):
        """
        Базовое окно настроек ПФ с некоторыми частоиспользуемыми контролами
        При наследовании от него нужно будет только переопределять
        функции сборки отдельных блоков.
        """

        def __init__(self, *args, **kwargs):
            super(BaseReportWindow, self).__init__(*args, **kwargs)
            self.title = u""
            self.template_globals = "BaseReportWindow.js"
            self.height = 480
            self.width = 380
            self.minimizable = False
            self.maximizable = False
            self.modal = True
            # Компонент формы
            self.form = ExtContainer()
            self.form.url = ''
            # Период
            self.p_cont = ExtContainer(layout='hbox', height=30,
                                       style={'padding': '2px'})
            self.ds_cont = ExtContainer(layout='form', flex=1, label_width=59,
                                        style={"padding-right": "5px"})
            self.period_since = ExtDictSelectField(
                anchor='100%',
                name='period_since',
                display_field='locale_period',
                value_field="id",
                label=u'Период с',
                trigger_action_all=True,
                ask_before_deleting=False,
                hide_trigger=False,
                hide_edit_trigger=True,
                hide_clear_trigger=True,
                hide_dict_select_trigger=True)
            self.period_since.pack = urls.get_pack("global-periods")
            self.ds_cont.items.append(self.period_since)

            self.du_cont = ExtContainer(layout='form', flex=1, label_width=59)
            self.period_until = ExtDictSelectField(
                anchor='100%',
                name='period_until',
                display_field='locale_period',
                value_field="id",
                label=u'по',
                trigger_action_all=True,
                ask_before_deleting=False,
                hide_trigger=False,
                hide_edit_trigger=True,
                hide_clear_trigger=True,
                hide_dict_select_trigger=True)
            self.period_until.pack = urls.get_pack("global-periods")
            self.du_cont.items.append(self.period_until)
            self.p_cont.items.extend([
                self.ds_cont,
                self.du_cont])

            # Поля
            self.print_all_sps = ExtCheckBox()
            self.print_all_sps.name = "print_all_sps"
            self.print_all_sps.label = u"Печатать по всему учреждению"

            self.print_ent_detail = ExtCheckBox()
            self.print_ent_detail.name = "print_ent_detail"
            self.print_ent_detail.label = u"Детализация по учреждениям"

            self.print_serv_detail = ExtCheckBox()
            self.print_serv_detail.name = "print_serv_detail"
            self.print_serv_detail.label = u"Детализация по услугам"

            self.print_state_serv = ExtCheckBox()
            self.print_state_serv.name = "print_state_serv"
            self.print_state_serv.label = u"По государственным услугам"

            self.print_paid_serv = ExtCheckBox()
            self.print_paid_serv.name = "print_paid_serv"
            self.print_paid_serv.label = u"По платным услугам"

            # Контейнер для них
            self.field_cont = ExtContainer(
                label_width=335,
                layout='form',
                style={'padding': '2px'}
            )

            # Грид с группами
            self.sp_grid = ExtObjectGrid(sm=ExtGridCheckBoxSelModel())
            self.sp_grid.paging_bar = False
            self.sp_grid.add_column(header=u"Группа", data_index="name")
            self.sp_grid.action_data = urls.get_url("report-group-rows")
            self.sp_grid.name = 'servicepoint_id'
            self.sp_grid.height = 150
            # Контейнер для него
            self.sp_cont = ExtContainer(style={'padding': '2px'})
            self.sp_cont.items.extend([self.sp_grid])

            # Грид с детьми
            self.knd_grid = ExtObjectGrid(sm=ExtGridCheckBoxSelModel())
            self.knd_grid.paging_bar = False
            self.knd_grid.add_column(header=u"Ребенок", data_index="name")
            self.knd_grid.action_data = urls.get_url("report-kinder-rows")
            self.knd_grid.name = 'kinder_id'
            self.knd_grid.height = 150
            # Контейнер для него
            self.knd_cont = ExtContainer(style={'padding': '2px'})
            self.knd_cont.items.extend([self.knd_grid])

            # Грид с районами
            self.rayon_grid = ExtObjectGrid(sm=ExtGridCheckBoxSelModel())
            self.rayon_grid.paging_bar = False
            self.rayon_grid.add_column(header=u"Районы", data_index="name")
            self.rayon_grid.action_data = urls.get_url("report-rayon-rows")
            self.rayon_grid.name = 'rayon_id'
            self.rayon_grid.height = 150
            # Контейнер для него
            self.rayon_cont = ExtContainer(style={'padding': '2px'})
            self.rayon_cont.items.extend([self.rayon_grid])

            self.items.append(self.form)

            # Описание кнопок
            self.print_btn = ExtButton()
            self.print_btn.name = 'print_btn'
            self.print_btn.text = u"Печатать"
            self.print_btn.handler = 'okHandler'
            self.cancel_btn = ExtButton()
            self.cancel_btn.name = 'cancel_btn'
            self.cancel_btn.text = u"Закрыть"
            self.cancel_btn.handler = 'closeHandler'
            # Добавление кнопок в окно
            self.buttons.extend([self.print_btn, self.cancel_btn])


Файл логики на javascript - *~/templates/ui-js/BaseReportWindow.js*:

    ::javascript


    var win = Ext.getCmp("{{component.client_id}}");
    var form = Ext.getCmp("{{component.form.client_id}}");
    var sp_grid = Ext.getCmp("{{component.sp_grid.client_id}}");
    var knd_grid = Ext.getCmp("{{component.knd_grid.client_id}}");
    var rayon_grid = Ext.getCmp("{{component.rayon_grid.client_id}}");
    var period_since = Ext.getCmp("{{component.period_since.client_id}}");
    var period_until = Ext.getCmp("{{component.period_until.client_id}}");
    var print_all_sps = Ext.getCmp("{{component.print_all_sps.client_id}}");
    var print_state_serv = Ext.getCmp("{{component.print_state_serv.client_id}}");
    var print_paid_serv = Ext.getCmp("{{component.print_paid_serv.client_id}}");
    var print_serv_detail = Ext.getCmp("{{component.print_serv_detail.client_id}}");
    var print_ent_detail = Ext.getCmp("{{component.print_ent_detail.client_id}}");
    var service = Ext.getCmp("{{component.service_field.client_id}}");


    function collectGridIds(grid){
        var selIds = [];
        var selRecords = grid.getSelectionModel().getSelections();
        for (var j=0;j<selRecords.length;j++){
            selIds.push(selRecords[j].id);
        }
        return Ext.util.JSON.encode(selIds);
    }

    if (print_all_sps){
        print_all_sps.on("check", function(){
            if (print_all_sps.getValue()){
                if (sp_grid) sp_grid.getSelectionModel().selectAll();
            } else {
                if (sp_grid) sp_grid.getSelectionModel().clearSelections();
            }
        });
    }

    if (sp_grid){
        sp_grid.getSelectionModel().on('selectionchange',function(){
            if (knd_grid) knd_grid.getStore().load();
        });
    }

    if (knd_grid){
        knd_grid.getStore().on('beforeload', function(request){
            request.baseParams["sps"] = collectGridIds(sp_grid);
        });
    }

    //метод получения всех значений формы
    function getSubmitValues(form){
        if (!form) return {};
        if (knd_grid && !knd_grid.getSelectionModel().hasSelection()) {
            Ext.Msg.show({
                 title: 'Внимание!',
                 msg: 'Не выбрано ни одного ребенка для печати!',
                 icon: Ext.MessageBox.WARNING,
                 buttons: Ext.MessageBox.OK
             });
            return false;
        }
        if (rayon_grid && !rayon_grid.getSelectionModel().hasSelection()) {
            Ext.Msg.show({
                 title: 'Внимание!',
                 msg: 'Не выбрано ни одного района для печати!',
                 icon: Ext.MessageBox.WARNING,
                 buttons: Ext.MessageBox.OK
             });
            return false;
        }
        var params = {};
        if (period_since) params['period_since'] = period_since.getValue();
        if (period_until) params['period_until'] = period_until.getValue();
        if (print_serv_detail) params['print_serv_detail'] = print_serv_detail.getValue();
        if (print_ent_detail) params['print_ent_detail'] = print_ent_detail.getValue();
        if (print_paid_serv) params['print_paid_serv'] = print_paid_serv.getValue();
        if (print_state_serv) params['print_state_serv'] = print_state_serv.getValue();
        if (print_all_sps) params['print_all_sps'] = print_all_sps.getValue();
        if (sp_grid) params["sps"] = collectGridIds(sp_grid);
        if (knd_grid) params["knds"] = collectGridIds(knd_grid);
        if (rayon_grid) params["rayons"] = collectGridIds(rayon_grid);
        if (service) params["service"] = service.getValue();
        return params;
    }

    function okHandler(btn, arguments){
        var formParams = getSubmitValues(form);
        if (!formParams) return;
        var mask = new Ext.LoadMask(win.body, {msg:'Подготовка формы...'});
        var params = Ext.apply(win.actionContextJson, formParams || {});
        mask.show();
        btn.disable();
        Ext.Ajax.request({
             url:"{{component.form.url}}",
             params: params,
             success: function(response, opts){
                 mask.hide();
                 smart_eval(response.responseText);
                 win.close();
             },
             failure: function(response, opts){
                 uiAjaxFailMessage.apply(win, arguments);
             },
             timeout: 600000 //10минут
         });
    }

    function closeHandler(btn, arguments){
        win.close();
    }



### <a name="example-in-3">Как решается эта же задача на версии m3 3</a>

Реализация экшена ([diff](https://www.diffchecker.com/67gc3cfg)):

    ::python

    class CompensationInfoReportWindowAction(UIAction):
        """
        Вызов диалога "Информация о компенсации"
        """
        shortname = "compensation-info-report-window"
        url = '/%s' % shortname
        verbose_name = u'Окно настроек печати'

        get_ui = lambda self, request, context: ui.CompensationInfoReportWindow(self)

Реализация UI ([diff](https://www.diffchecker.com/77zencej)):

    ::python

    class CompensationInfoReportWindow(BaseReportWindow):
        """
        Окно настроек печати оборотки по районам
        """
        def __init__(self, parent=None):
            super(CompensationInfoReportWindow, self).__init__()
            self.submit_url = urls.get_url("compensation-info-report")
            self.title = parent.verbose_name if parent else u''

            # скорректируем высоту
            self.height = 370
            self.sp_grid.height = 290

            # соберем нужные блоки
            self.form.items.extend([
                self.sp_cont
            ])

Класс-наследник для UI *BaseReportWindow*,

- практически [не изменился](https://www.diffchecker.com/st0qdykz) за исключением появления *xtype*, удаления *template-globals*
- modal=True - уже не нужен, так как этот механизм заложен на уровне платформы
- используется нативное событие *close*, для кнопки "Выход".

        ::python

        class BaseReportWindow(ExtWindow):
            """
            Базовое окно настроек ПФ с некоторыми частоиспользуемыми контролами
            При наследовании от него нужно будет только переопределять
            функции сборки отдельных блоков.
            """
            _xtype = 'base-report-window'

            def __init__(self, *args, **kwargs):
                super(BaseReportWindow, self).__init__(*args, **kwargs)
                self.title = u""
                self.height = 480
                self.width = 380
                self.minimizable = False
                self.maximizable = False
                # self.modal = True
                # Компонент формы
                self.form = ExtContainer()
                self.form.url = ''
                # Период
                self.p_cont = ExtContainer(layout='hbox', height=30,
                                           style={'padding': '5px'}
                )
                self.ds_cont = ExtContainer(layout='form', flex=1, label_width=59,
                                            style={"padding-right": "5px"}
                )
                self.period_since = ExtDictSelectField(
                    anchor='100%',
                    name='period_since',
                    display_field='locale_period',
                    value_field="id",
                    label=u'Период с',
                    trigger_action=ExtDictSelectField.ALL,
                    ask_before_deleting=False,
                    hide_trigger=False,
                    hide_edit_trigger=True,
                    hide_clear_trigger=True,
                    hide_dict_select_trigger=True)
                self.period_since.pack = urls.get_pack("global-periods")
                self.ds_cont.items.append(self.period_since)

                self.du_cont = ExtContainer(layout='form', flex=1, label_width=59)
                self.period_until = ExtDictSelectField(
                    anchor='100%',
                    name='period_until',
                    display_field='locale_period',
                    value_field="id",
                    label=u'по',
                    trigger_action_all=True,
                    ask_before_deleting=False,
                    hide_trigger=False,
                    hide_edit_trigger=True,
                    hide_clear_trigger=True,
                    hide_dict_select_trigger=True)
                self.period_until.pack = urls.get_pack("global-periods")
                self.du_cont.items.append(self.period_until)
                self.p_cont.items.extend([
                    self.ds_cont,
                    self.du_cont])

                # Поля
                self.print_all_sps = ExtCheckBox()
                self.print_all_sps.name = "print_all_sps"
                self.print_all_sps.label = u"Печатать по всему учреждению"

                self.print_ent_detail = ExtCheckBox()
                self.print_ent_detail.name = "print_ent_detail"
                self.print_ent_detail.label = u"Детализация по учреждениям"

                self.print_serv_detail = ExtCheckBox()
                self.print_serv_detail.name = "print_serv_detail"
                self.print_serv_detail.label = u"Детализация по услугам"

                self.print_state_serv = ExtCheckBox()
                self.print_state_serv.name = "print_state_serv"
                self.print_state_serv.label = u"По государственным услугам"

                self.print_paid_serv = ExtCheckBox()
                self.print_paid_serv.name = "print_paid_serv"
                self.print_paid_serv.label = u"По платным услугам"

                # Контейнер для них
                self.field_cont = ExtContainer(
                    label_width=335,
                    layout='form',
                    style={'padding': '5px'}
                )

                # Грид с группами
                self.sp_grid = ExtObjectGrid(sm=ExtGridCheckBoxSelModel())
                self.sp_grid.allow_paging = False
                self.sp_grid.add_column(header=u"Группа", data_index="name")
                self.sp_grid.action_data = urls.get_url("report-group-rows")
                self.sp_grid.name = 'servicepoint_id'
                self.sp_grid.height = 150
                # Контейнер для него
                self.sp_cont = ExtContainer()
                self.sp_cont.items.extend([self.sp_grid])

                # Грид с детьми
                self.knd_grid = ExtObjectGrid(sm=ExtGridCheckBoxSelModel())
                self.knd_grid.allow_paging = False
                self.knd_grid.add_column(header=u"Ребенок", data_index="name")
                self.knd_grid.action_data = urls.get_url("report-kinder-rows")
                self.knd_grid.name = 'kinder_id'
                self.knd_grid.height = 150
                # Контейнер для него
                self.knd_cont = ExtContainer()
                self.knd_cont.items.extend([self.knd_grid])

                # Грид с районами
                self.rayon_grid = ExtObjectGrid(sm=ExtGridCheckBoxSelModel())
                self.rayon_grid.allow_paging = False
                self.rayon_grid.add_column(header=u"Районы", data_index="name")
                self.rayon_grid.action_data = urls.get_url("report-rayon-rows")
                self.rayon_grid.name = 'rayon_id'
                self.rayon_grid.height = 150
                # Контейнер для него
                self.rayon_cont = ExtContainer()
                self.rayon_cont.items.extend([self.rayon_grid])

                self.items.append(self.form)

                # Описание кнопок
                self.print_btn = ExtButton()
                self.print_btn.name = 'print_btn'
                self.print_btn.text = u"Печатать"
                self.print_btn.handler = 'okHandler'
                self.cancel_btn = ExtButton()
                self.cancel_btn.name = 'cancel_btn'
                self.cancel_btn.text = u"Закрыть"
                self.cancel_btn.handler = 'close'
                # Добавление кнопок в окно
                self.buttons.extend([self.print_btn, self.cancel_btn])

Файл логики на javascript - *~/static/js/base-report-window.js*


    ::javascript

    Ext.define('Ext.paidserv.BaseReportWindow', {
        extend: 'Ext.m3.Window',
        xtype: 'base-report-window',
        initComponent: function () {
            this.callParent();

            this.form = this.findByItemId('form');
            this.sp_grid = this.findByItemId('sp_grid');
            this.knd_grid = this.findByItemId('knd_grid');
            this.rayon_grid = this.findByItemId('rayon_grid');
            this.period_since = this.findByItemId('period_since');
            this.period_until = this.findByItemId('period_until');
            this.print_all_sps = this.findByItemId('print_all_sps');
            this.print_state_serv = this.findByItemId('print_state_serv');
            this.print_paid_serv = this.findByItemId('print_paid_serv');
            this.print_serv_detail = this.findByItemId('print_serv_detail');
            this.print_ent_detail = this.findByItemId('print_ent_detail');
            this.service = this.findByItemId('service');


            if (this.print_all_sps) {
                this.print_all_sps.on("check", function () {
                    if (this.print_all_sps.getValue()) {
                        if (this.sp_grid) {
                            this.sp_grid.getSelectionModel().selectAll();
                        }
                    } else {
                        if (this.sp_grid) {
                            this.sp_grid.getSelectionModel().clearSelections();
                        }
                    }
                }.bind(this));
            }

            if (this.sp_grid) {
                this.sp_grid.getSelectionModel().on('selectionchange', function () {
                    if (this.knd_grid) {
                        this.knd_grid.getStore().load();
                    }
                }, this);
            }

            if (this.knd_grid) {
                this.knd_grid.getStore().on('beforeload', function (request) {
                    request.baseParams["sps"] = this.collectGridIds(this.sp_grid);
                }, this);
            }

        },

        collectGridIds: function (grid) {
            var selIds = [],
                selRecords = grid.getSelectionModel().getSelections();
            for (var j = 0; j < selRecords.length; j++) {
                selIds.push(selRecords[j].id);
            }
            return Ext.encode(selIds);
        },

        //метод получения всех значений формы
        getSubmitValues: function (form) {
            if (!form) return {};
            if (this.knd_grid && !this.knd_grid.getSelectionModel().hasSelection()) {
                Ext.Msg.show({
                    title: 'Внимание!',
                    msg: 'Не выбрано ни одного ребенка для печати!',
                    icon: Ext.MessageBox.WARNING,
                    buttons: Ext.MessageBox.OK
                });
                return false;
            }
            if (this.rayon_grid && !this.rayon_grid.getSelectionModel().hasSelection()) {
                Ext.Msg.show({
                    title: 'Внимание!',
                    msg: 'Не выбрано ни одного района для печати!',
                    icon: Ext.MessageBox.WARNING,
                    buttons: Ext.MessageBox.OK
                });
                return false;
            }
            var params = {};
            if (this.period_since) {
                params['period_since'] = this.period_since.getValue();
            }
            if (this.period_until) {
                params['period_until'] = this.period_until.getValue();
            }
            if (this.print_serv_detail) {
                params['print_serv_detail'] = this.print_serv_detail.getValue();
            }
            if (this.print_ent_detail) {
                params['print_ent_detail'] = this.print_ent_detail.getValue();
            }
            if (this.print_paid_serv) {
                params['print_paid_serv'] = this.print_paid_serv.getValue();
            }
            if (this.print_state_serv) {
                params['print_state_serv'] = this.print_state_serv.getValue();
            }
            if (this.print_all_sps) {
                params['print_all_sps'] = this.print_all_sps.getValue();
            }
            if (this.sp_grid) {
                params["sps"] = this.collectGridIds(this.sp_grid);
            }
            if (this.knd_grid) {
                params["knds"] = this.collectGridIds(this.knd_grid);
            }
            if (this.rayon_grid) {
                params["rayons"] = this.collectGridIds(this.rayon_grid);
            }
            if (this.service) {
                params["service"] = this.service.getValue();
            }
            return params;
        },

        okHandler: function (btn, arguments) {
            var formParams = this.getSubmitValues(this.form);
            if (!formParams) {
                return;
            }

            UI.callAction.call(this, {
                method: 'POST',
                url: this.submitUrl,
                params: Ext.apply(formParams || {}, this.getContext()),
                success: this.close.createDelegate(this),
                failure: uiAjaxFailMessage,
                timeout: 600000 // 10 минут
            });

        },

        bind: function (data) {
            this.submitUrl = data['submit_url'];
        }
    });


Как видим отличия как правило заключаются в качественно другом представлении javascript-кода.