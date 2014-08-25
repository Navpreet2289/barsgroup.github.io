<!--
.. title: Master-Detail окно
.. slug: master-detail-okno
.. date: 2014-08-25 11:14:31 UTC+04:00
.. tags:
.. link:
.. description:
.. type: text
-->

### Пример реализации Master-Detail окна.

*MD*-представление теперь реализуется двумя паками - для master-грида(дерева) и для detail-грида соответственно.

Master-pack имеет вид:

    ::python
    class ArticlePack(objectpack.ObjectPack):

        model = Article

        # list_window указывать не нужно, т.к. экшн этим занимается сам

        def __init__(self):
            super(ArticlePack, self).__init__()
            self.replace_action(
                'list_window_action', ArticleListWindowAction())


Экшн, отвечающий за отображение MD-окна имеет вид:

    ::python
    class GarageMDWindowAction(objectpack.actions.MasterDetailWindowAction):

        # Можно указать класс окна - потомка от MasterDetailWindow
        #window_clz = ui.ArticleListWindow

        @property
        def detail_pack(self):
            # возвращается экземпляр пака для detail-грида
            return ControllerCache.find(CommentPack)

        # здесь можно модифицировать окно перед его рендерингом,
        # однако значительные изменения лучше производить в окне-наследнике
        def create_window(self):
            super(GarageMDWindowAction, self).create_window()
            self.win.title = self.parent.title

Типичная задача, решаемая наследованием типового окна - переопределение класса master grid. Например, в том случае, когда мастером должно быть дерево:

    ::python
    class ArticleListWindow(objectpack.ui.MasterDetailWindow):

        master_grid_clz = ExtObjectTree
