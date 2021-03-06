---
layout: post
title: Dashboard代码分析
tags: flask web
category: it
---

{% raw %}


# UI框架

##初始化
我们先从服务启动脚本开始，把源码跟一遍，理解服务初始化过程

* `server`

        adminportal = create_app(app_name = CLOUD_APP_NAME, modules = CLOUD_MODULES)
        CloudPortal.init_app(adminportal)
        adminportal.run(host='0.0.0.0', port=5000)

    这个是启动文件，Flask-Admin的标准写法；  
    `create_app()`注册一些信息（具体干了什么和参数的意义都可以先不深究），最终返回一个`Flask`对象（见`portal/application.py`）；  
    `init_app()`一看便可猜到`CloudPortal`应该是一个`Admin`的封装；  
    最后启动服务

* `portal/cloud/dashboard.py`

        class OverviewPanels(object):
            category = _("Overview")
            panels = ("overview",)
        
        class ManageComputePanels(object):
            category = _("Manage Compute")
            panels = ("instances",
                      "volumes",
                      "images_snapshots",
                      "access_security",)
        
        class TopologyPanels(object):
            panels = ("topology",)
            
        class Settings(object):
            panels = ("user",
                      "auth",)
        
        class Project(Dashboard):
            default_panel = OverviewPanels
            panels = (ManageComputePanels,
                      TopologyPanels,
                      Settings)
        
        CloudPortal = Project().register()

    这里一堆相同结构的class，看不懂，也就先不管；先搞清楚`CloudPortal`，所以再往下跟；

* `portal/cloud/base.py`

        class UserPortal(Admin):
            def __init__(self, **kwargs):
                super(UserPortal, self).__init__(**kwargs)
                self.base_template = 'base.html'

        
            def _add_view_to_menu(self, view):
                """
                Add a view to the menu tree
        
                :param view: View to add
                """
                if view.category:
                    category = self._menu_categories.get(view.category)
        
                    if category is None:
                        category = PortalMenuItem(view.category, view.menu_icon)
                        self._menu_categories[view.category] = category
                        self._menu.append(category)
        
                    category.add_child(PortalMenuItem(view.name, view.menu_icon, view))
                else:
                    self._menu.append(PortalMenuItem(view.name, view.menu_icon, view))

        
        class Dashboard(object):
            panels = []
            default_panel = None
        
            def __init__(self):
                self.registry_views = []

        
            def _autodiscovery(self, panel):
                package = import_module('portal.cloud.%s' % panel)
                views_to_register = package.views_to_register
                for view in views_to_register:
                    self.registry_views.append(view)

        
            def register(self):
                # Register Index View
                for panel in self.default_panel.panels:
                    self._autodiscovery(panel)
                    index_view = self.registry_views.pop()
                    userportal = UserPortal(index_view=index_view())
        
                # Register Other Views
                for panel in self.panels:
                    for panel_to_register in panel.panels:
                        category = getattr(panel, "category", None)
                        self._autodiscovery(panel_to_register)
                        length = len(self.registry_views)
                        for i in range(length):
                            view = self.registry_views.pop()
                            userportal.add_view(view(category=category))
        
                return userportal

    `Dashboard`的主要工作就是`add_view`，然后返回`userportal`（一个`Admin`的封装）；  
    接着我们调试看看`index_view()`是什么以及`add_view()`的参数：

        index_view: <portal.cloud.overview.views.Overview object at 0x3126ad0>

        userportal.add_view(<class 'portal.cloud.instances.views.Instances'>(category='Manage Compute'))
        userportal.add_view(<class 'portal.cloud.volumes.views.Volumes'>(category='Manage Compute'))
        userportal.add_view(<class 'portal.cloud.images_snapshots.views.Images'>(category='Manage Compute'))
        userportal.add_view(<class 'portal.cloud.access_security.api_access.views.ApiAccess'>(category='Manage Compute'))
        userportal.add_view(<class 'portal.cloud.access_security.floating_ips.views.FloatingIps'>(category='Manage Compute'))
        userportal.add_view(<class 'portal.cloud.access_security.keypairs.views.Keypairs'>(category='Manage Compute'))
        userportal.add_view(<class 'portal.cloud.access_security.security_groups.views.SecurityGroups'>(category='Manage Compute'))
        userportal.add_view(<class 'portal.cloud.access_security.views.AccessSecurity'>(category='Manage Compute'))
        userportal.add_view(<class 'portal.cloud.topology.views.Topology'>(category=None))
        userportal.add_view(<class 'portal.cloud.user.views.User'>(category=None))
        userportal.add_view(<class 'portal.cloud.auth.views.Auth'>(category=None))

    结合首页左侧的菜单，我们现在可以理解`dashboard.py`中的类意义，就是菜单结构及对应的页面（panels）；  
    跟着调试一下`_autodiscovery()`就可知道模块的路径：

        import_module('portal.cloud.overview')
        import_module('portal.cloud.instances')
        import_module('portal.cloud.volumes')
        import_module('portal.cloud.images_snapshots')
        import_module('portal.cloud.access_security')
        import_module('portal.cloud.topology')
        import_module('portal.cloud.user')
        import_module('portal.cloud.auth')

    接下来分析`Instances`页面的实现；

* `portal/cloud/instances/__init__.py`

        from .views import Instances
        views_to_register = (Instances,)

    每个模块的`__init__.py`都是差不多的，就是告诉你最终的view在哪儿，接着看`views.py`；

* `portal/cloud/instances/views.py`

        from .base import InstanceBaseView

        class Instances(InstanceBaseView, Table):
        
            @expose('/')
            def index(self):
                ...
        
            @expose('/details/')
            def details(self):
                ...
        
            @expose('/edit/', methods=('GET', 'POST'))
            def edit(self):
                ...
        
            @expose('/launch/', methods=('GET', 'POST'))
            def launch(self):
                ...
        
            @expose('/terminate/', methods=('GET', 'POST'))
            def terminate(self):
                ...

    这里很明显就是页面处理的部分，`Instances`应该能猜到就是`BaseView`的子类；跟进看`base.py`，先把封装逐层剥清楚；

* `portal/cloud/instances/base.py`

        from portal.cloud.base import CloudBaseView
        
        class InstanceBaseView(CloudBaseView):
        
            index_template = 'cloud/instances/index.html'
            workflow_template = 'cloud/workflow.html'
            details_template = 'cloud/instances/details.html' 
        
            def __init__(self, name=None, category=None, endpoint=None, url=None):
        
                self.init_actions()
                super(InstanceBaseView, self).__init__(name, category, endpoint, url)

                self.url = 'instances'
                self.menu_icon = 'iconfa-th-list'
        
            @expose('/action/', methods=('POST',))
            def action_view(self):
                return self.handle_action()
        
            @action('launch', lazy_gettext('Launch Instance'),
                    classes="btn", icon_class="icon-plus")
            def action_launch_instance(self, items):
                pass
        
            @action('terminate', lazy_gettext('Terminate Instance'), ajax_modal=False,
                     classes="btn btn-primary", icon_class="icon-trash icon-white")
            def action_terminate_instance(self, items):
                pass

    这里有些`ActionsMixin`的代码，以及模板路径、url、菜单图标；再往下跟

* `portal/cloud/base.py`

        def action(name, text, classes, icon_class,
                   ajax_modal=True, confirmation=None):

            def wrap(f):
                f._action = (name, text, classes, icon_class,
                             ajax_modal, confirmation)
                return f
        
            return wrap
        
        class CloudBaseView(BaseView, ActionsMixin):
        
            def __init__(self, name=None, category=None, endpoint=None, url=None,
                         static_folder=None, static_url_path=None):
            
                class_name = self.__class__.__name__.lower()
                
                self.url = url or '/%s' % 'cloud'
                self.static_folder = 'static'
        
                super(CloudBaseView, self).__init__(name, category, endpoint, 
                                                    self.url,
                                                    self.static_folder,
                                                    static_url_path)


            def create_blueprint(self, admin):

                self.blueprint = super(CloudBaseView, self).create_blueprint(admin)
        
                # Update blueprint root_path, so that we can use `url_for(.static)`
                self.blueprint.root_path = os.path.dirname(os.path.abspath(__name__))
        
                return self.blueprint


            def init_actions(self):

                self._actions = []
                self._actions_data = {}
        
                for p in dir(self):
                    attr = tools.get_dict_attr(self, p)
        
                    if hasattr(attr, '_action'):
                        name, text, classes, icon_class, ajax_modal, desc= attr._action
        
                        self._actions.append((name, text, classes, icon_class,
                                              ajax_modal))
        
                        self._actions_data[name] = (getattr(self, p), text, desc,
                                                    classes, icon_class, ajax_modal)

        
            def get_actions_list(self):

                actions = []
                actions_confirmation = {}
        
                for act in self._actions:
                    name, text, classes, icon_class, ajax_modal = act
        
                    if self.is_action_allowed(name):
                        actions.append((name, text_type(text), classes, icon_class,
                                        ajax_modal))
        
                        confirmation = self._actions_data[name][2]
                        if confirmation:
                            actions_confirmation[name] = text_type(confirmation)
                return actions, actions_confirmation

        
        class PortalMenuItem(MenuItem):

            def __init__(self, name, menu_icon, view=None):
                super(PortalMenuItem, self).__init__(name=name, view=view)
                self.menu_icon = menu_icon

    跟action相关的代码，都是为了扩展ActionsMixin，使它能支持menu icon和ajax_modal，暂时可忽略；  
    先来总结一下封装情况：

        BaseView -> CloudBaseView -> InstanceBaseView -> Instances

    封装后能更模块化的处理不同页面的URL  
    `CloudBaseView` (portal/cloud/base.py)： 将默认的URL前缀由'/admin'变成'/cloud'  
    `InstanceBaseView` (portal/cloud/instances/base.py)： `self.url = 'instances'`设置当前模块的URL，使instances页面的URL变成`/cloud/instances/`  
    `Instances` (portal/cloud/instances/views.py)： 都是页面处理方法，比如页面`/cloud/instances/details/`  

##页面：/cloud/instances/

接下来我们从页面角度分析，先看`/cloud/instances/`页面，处理函数是`views.py`中的`index()`：

* `cloud/instances/views.py`

        @expose('/')
        def index(self):

            self.columns = [_("Instance Name"), _("IP Address"), _("Size"),
                            _("Keypair"), _("Status"), _("Task"),
                            _("Power State"), _("Action")]
                            
            # Actions
            actions, actions_confirmation = self.get_actions_list()
    
            # For Get Request
            request = {'action':'list_instance', 'body':'{}'}
            dispatcher = Dispatcher(request)
            resp_id = dispatcher.sendRequest()['requestId']
    
            return self.render(self.index_template,
                               resp_id = resp_id,
                               columns=self.columns,
                               actions=actions,
                               actions_confirmation=actions_confirmation)

    把参数打印出来看看：

        # 调试信息
        actions: [('launch', u'Launch Instance', 'btn', 'icon-plus', True), ('terminate', u'Terminate Instance', 'btn btn-primary', 'icon-trash icon-white', False)]
        actions_confirmation: {}
        resp_id: 20131221221124e99cd871e76d4355ab44fbd7f948d0d

    `actions`的结构可以参考`get_actions_list()`的返回值：`(name, text, class, icon_class, ajax_modal)`  
    `actions_confirmation`是弹框确认信息，目前为空  
    `resp_id`看不懂，先不管  
    `index_template = 'cloud/instances/index.html'`在`base.py`中定义

* `cloud/instances/index.html`

        # 省略无关代码
        {% extends 'cloud/instances/layout.html' %}
        {% import 'flask_admin/tables.html' as tableslib with context %}

        {% block extra_javascript %}
            <script type="text/javascript" src="{{ url_for('.static', filename='js/jquery.dataTables.min.js') }}"></script>
            <script type="text/javascript" src="{{ url_for('.static', filename='js/userportal/getresponse.js') }}"></script>
        {% endblock %}

        {% block content %}
            {{ tableslib.render_table() }}
        {% endblock %}

* `cloud/instances/layout.html`

    里面就是一些title文本：

        {% extends 'cloud/layout.html' %}
        {% block pageicon %}iconfa-table{% endblock %}
        {% block smalltitle %} Virtual Machines {% endblock %}
        {% block bigtitle %} Instances Overview{% endblock %}

    模板继承关系如下：

        cloud/instances/index.html
        cloud/instances/layout.html
        cloud/layout.html
        base.html

* `flask_admin/tables.html`

    表格显示代码就在这里面，只有一个空的`table`：

        # 省略无关代码
        {% macro render_table() %}
            <table id="dyntable" class="dataTable table table-bordered responsive" data-resqid="{{ resp_id }}">
              <tbody>
              </tbody>
            </table>
        {% endmacro %}

* `js/userportal/getresponse.js`

    数据由JS自动填充（目前是死数据）

        jQuery(document).ready(function(){
          function fnClickAddRow() {
            jQuery('#dyntable').dataTable().fnAddData([
            [
                '<span class="center"><div id="uniform-undefined" class="checker"><span><input type="checkbox" /></span></div></span>',
                "Ubuntu 12.04",
                "10.196.5.7",
                "m1.tiny | 512MB RAM | 2 VCPUs | 0 Disk",
                "key01",
                "Active",
                "None",
                "Running",
                '<div class="btn-group "><button class="btn">Create Snapshot</button><button data-toggle="dropdown" class="btn btn-info dropdown-toggle">Action <span class="caret"></span></button><ul class="dropdown-menu"><li><a href="#">Associate Floting IP</a></li><li><a href="#">Disassociate Floting IP</a></li><li><a href="#">Edit Instance</a></li><li><a href="#">Edit Security Groups</a></li><li><a href="#">Console</a></li><li><a href="#">Boot Log</a></li><li><a href="#">Reboot</a></li><li><a href="#">Shutdown</a></li><li class="divider"></li><li><a href="#">Terminate</a></li></ul></div>' ],
            ]);
          }
        });

    表格显示使用`dataTables`插件，使用方法请参考相关文档，简单来说就是将要显示的数据全部传进去，然后由它负责显示数据清单以及翻页处理等等；  
    按钮` Launch Instance`和` Terminate Instance`的显示方法请参考`Flask-Admin`的`ActionsMixin`模块；


##页面：/cloud/instances/launch/

接下来，分析按钮`Launch Instance`和点击后出现的弹框

* 按钮+模态框

    页面部分代码如下，目前只需要关注两个按钮和一个空的`bootstrap`模态框：

        <p class="pull-right">
          <a data-toggle="modal" data-target="#myModal" href="/cloud/instances/launch/" class="btn"><i class="icon-plus"></i> Launch Instance</a>
          <a href="/cloud/instances/terminate/" class="btn btn-primary"><i class="icon-trash icon-white"></i> Terminate Instance</a>
        </p>
        
        <div id="myModal" class="modal hide fade in"></div>

* `cloud/instances/index.html`

        // ajax modal
        jQuery("a[data-toggle=modal]").click(function() {
          var target, url;
          target = jQuery(this).attr('data-target');
          url = jQuery(this).attr('href') + ' #mymodal';
          return jQuery(target).load(url, function(){
            jQuery.getScript("{{ url_for('.static', filename='js/forms.js') }}");
            jQuery.getScript("{{ url_for('.static', filename='js/userportal/wizard.js') }}");
            })
        });

    点击`Launch Instance`按钮后，实际执行的jQuery代码如下：
    
        jQuery('#myModal').load('/cloud/instances/launch/ #mymodal', function(){...})

    意思是将`/cloud/instances/launch/`页面的`mymodal`部分加载到`id="myModal"`的模态框里面，所以弹框也可以通过URL以页面形式访问；

* `cloud/instances/views.py`

    `/cloud/instances/launch/`页面的请求由如下方法处理：

        from .workflow import UpdateInstanceInfo, LaunchInstance

        @expose('/launch/', methods=('GET', 'POST'))
        def launch(self):

            return_url = url_for('.index')
    
            form = LaunchInstance(helpers.get_form_data())
            workflow = form.workflow()
    
            # For POST Resquest
            if helpers.validate_form_on_submit(form):
                return redirect(return_url)
    
            # For GET Request
            request = {'action':'list_image', 'body':'{}'}
            dispatcher = Dispatcher(request)
            resp_id = dispatcher.sendRequest()['requestId']
            ajax_request = {'image_id': resp_id}
            
            return self.render(self.workflow_template, form=form,
                               workflow=workflow, ajax_request=ajax_request)

    接下来，先了解参数中的`workflow`和`form`；

* `cloud/instances/workflow.py`

        from .forms import InstanceInfo, InstanceDetails, InstanceAccessControls, \
            InstanceNetworing, InstancePostCreation
        
        class LaunchInstance(InstanceDetails, InstanceAccessControls,
                             InstanceNetworing, InstancePostCreation):
        
            def workflow(self):
                # Ajax modal header
                self.modal_header = _('Launch Instance')
        
                # Ajax modal finalize button
                self.finalize_button_name = _('Launch')
        
                # Ajax modal steps showing instance creation process
                self.steps = [_("Details"), _("Access & Security"),
                              _("Netwoking"), _("Post-Creation")]
        
                # Hidden field
                self.instance_details = InstanceDetails()
                self.instance_details.image_id.hidden = True
        
                # Sub forms for each step
                self.forms = [self.instance_details, InstanceAccessControls(),
                              InstanceNetworing(), InstancePostCreation()]
        
                return zip(self.forms, self.steps)

    `workflow()`返回的数据：

        # 调试信息
        zip: [(<portal.cloud.instances.forms.InstanceDetails object at 0x10641a510>, u'Details'),
              (<portal.cloud.instances.forms.InstanceAccessControls object at 0x10641a550>, u'Access & Security'),
              (<portal.cloud.instances.forms.InstanceNetworing object at 0x10641a950>, u'Netwoking'),
              (<portal.cloud.instances.forms.InstancePostCreation object at 0x10641ab10>, u'Post-Creation')]

    `zip()`的将两个List组合起来了；

* `cloud/instances/workflow.py`

    表单处理使用`WTForms`库和`Flask-Admin form`模块（使用方法请参考相关文档）

        from flask.ext.admin import form
        from wtforms import fields, validators
        
        class InstanceInfo(form.BaseForm):
        
            name = fields.TextField(_("Instance name"))
        
        class InstanceDetails(form.BaseForm):
            ...

        class InstanceAccessControls(form.BaseForm):
            ...

        class InstanceNetworing(form.BaseForm):
            ...

        class InstancePostCreation(form.BaseForm):
            ...

* `cloud/workflow.html`

    `workflow_template = 'cloud/workflow.html'`在`base.py`中定义；

        {% extends 'cloud/layout.html' %}
        {% import 'flask_admin/workflow_wizard.html' as lib with context %}

        {% block pageicon %} iconfa-table {% endblock %}
        {% block smalltitle %} Virtual Machines {% endblock %}
        {% block bigtitle %} Instaxxxxxxxxx:nces {% endblock %}

        {% block content %}
          {{ lib.render_workflow() }}
        {% endblock %}

* `flask_admin/workflow_wizard.html`

        # 省略部分
        {% macro render_workflow() %}
          <div id="mymodal">
            <form class="stdform" method="POST" action="#">
              ...
            </form>
          </div>
        {% endmacro %}

    这块代码就是模态框加载的`mymodal`部分页面，最终HTML如下：

        # 部分代码 
        <div id="mymodal">
          <form class="stdform" method="POST" action="#">
            <div class="modal-body">
              <div id="wizard3" class="wizard tabbedwizard">
                ...
              </div>
            </div>
          </form>
        </div>

* `js/userportal/wizard.js`

        jQuery(document).ready(function(){
            
          jQuery('#wizard3').smartWizard({onFinish: onFinishCallback});

          var current = 0;
          function onFinishCallback(){
            jQuery("#progress").show();
            jQuery(".buttonFinish").addClass('buttonDisabled');
            var progress = setInterval(function(){
              if(current >= 100){
                clearInterval(progress);
                jQuery('#myModal').modal('hide');
                fnClickAddRow();
                return false;
              }else{
                current ++;
                jQuery('#pbar').css('width', current + '%');    
              }
            },100);
          }
        
          function fnClickAddRow() {
            jQuery('#dyntable').dataTable().fnAddData([
            [   
                '<span class="center"><div id="uniform-undefined" class="checker"><span><input type="checkbox" /></span></div></span>',
                "CentOS 6.5",
                "10.196.5.8",
                "m1.tiny | 512MB RAM | 2 VCPUs | 0 Disk",
                "key01",
                "Active",
                "None",
                "Running",
                '<div class="btn-group"><button class="btn">Create Snapshot</button><button data-toggle="dropdown" class="btn btn-info dropdown-toggle">Action <span class="caret"></span></button><ul class="dropdown-menu"><li><a href="#">Associate Floting IP</a></li><li><a href="#">Disassociate Floting IP</a></li><li><a href="#">Edit Instance</a></li><li><a href="#">Edit Security Groups</a></li><li><a href="#">Console</a></li><li><a href="#">Boot Log</a></li><li><a href="#">Reboot</a></li><li><a href="#">Shutdown</a></li><li class="divider"></li><li><a href="#">Terminate</a></li></ul></div>' ],
            ]); 
            
          }
        });

    当依次执行完4个Step后，点击`Finish`按钮，出现进度条（假的），然后Table增加一条记录（假数据）；  
    更多请参考`jQuery smartWizard`插件

#异步调用

现在深入分析`Dispacher`异步调用机制

* `cloud/instances/views.py`

        @expose('/')
        def index(self):

            # For Get Request
            request = {'action':'list_instance', 'body':'{}'}
            dispatcher = Dispatcher(request)
            resp_id = dispatcher.sendRequest()['requestId']

* `portal/utils/dispatch/core.py`

        from portal.utils.actionProxy.core import KEY_ACTION, KEY_BODY

        KEY_RESULT = "kEyReSuLt"
        KEY_REQID = "requestId"
        VALUE_NOTFOUND = "notFound"
        VALUE_ARG_INVALID = "argumentsInvalid"

        class Dispatcher(object):
            def __init__(self, request, session_id=None):
                if session_id == None:
                    from flask import session
                    sessionId = session['session_id']
                else:
                    sessionId = session_id
                self.sessionMgmt = SessionMgmt(sessionId)
                self.request = request
                self.actProxy = ActionProxy()
                self.actionMgmt = ActionMgmt()
                self.respMgmt = RespMgmt()
        
            def sendRequest(self):
                if not self._checkRequest([KEY_ACTION, KEY_BODY]):
                    return {KEY_REQID:VALUE_ARG_INVALID}
                reqId = self._genReqId()
                self.actionMgmt.addActionRecord(self.request, self.sessionMgmt, reqId)
                greenth = eventlet.spawn(self._background, reqId)
                greenth.link(self._callback)
                resp = {KEY_REQID:reqId}
                return resp
        
            def getResponse(self):
                if not self._checkRequest([KEY_REQID]):
                    return {KEY_REQID:VALUE_ARG_INVALID}
                reqId = self.request.get(KEY_REQID, None)
                resp = self.respMgmt.getResponse(reqId)
                if resp == None:
                    resp = {KEY_REQID:VALUE_NOTFOUND}
                else:
                    self.respMgmt.delResponse(reqId)
                return resp
        
            def _checkRequest(self, keyList):
                if self.request == None:
                    return False
                for key in keyList:
                    if not self.request.has_key(key):
                        return False
                return True
        
            def _genReqId(self):
                dt = datetime.now()
                while True:
                    reqId = dt.strftime('%Y%m%d%H%M%S') + uuid.uuid4().hex
                    if not self.respMgmt.isExist(reqId):
                        break
                return reqId
        
            def _background(self, reqId):
                """ do something in the background """
                resp = self.actProxy.performAction(self.request,
                                                   self.sessionMgmt.getSession())
                return reqId, resp
        
            def _callback(self, gt, *args, **kwargs):
                reqId, resp = gt.wait()
                self.respMgmt.addResponse(reqId, resp)

    变量：

        KEY_ACTION = 'action'
        KEY_BODY = 'body'

    session：

        # 调试信息
        session: <SecureCookieSession {u'session_id': 'TestSession'}>

    各种初始化：

        GLOBAL_DICT_SESSION = {}
        class SessionMgmt(object):
            def __init__(self, sessionId=None):
                self.sessionDict = GLOBAL_DICT_SESSION
                self.sessionId = sessionId

        class ActionProxy(object):
            def __init__(self):
                self.actionMgmt = ActionMgmt()
                self.actMan = FakeActManager()

        class ActionMgmt(object):
            def __init__(self):
                pass
        
        class FakeActManager():
            def __init__(self):
                pass
        
        GLOBAL_DICT = {}
        class RespMgmt(object):
            def __init__(self):
                self.respDict = GLOBAL_DICT

    `Dispatcher(request)`：

     * 对于`sendRequest()`：`request = {'action': xxx, 'body': xxx}`
     * 对于`getResponse()`：`request = {'requestId': xxx}`

    `sendRequest()`：

    1. 检查参数`request`格式合法性
    2. 产生一个随机ID
    3. `addActionRecord()`（暂无实际意义）
    4. 创建线程，然后`performAction()`（实际调用`action_xxx()`获取数据）；线程结束前执行`addResponse()`（保存数据）
    5. 返回`{'requestId': ID}`  
    
    &nbsp;
    
        # ActionMgmt
        def addActionRecord(self, request, session, reqId):
            return True

        # ActionProxy 
        def performAction(self, request, session):
            action = request.get(KEY_ACTION, None)
            ret = self.actMan.doAction(action, session)
            return ret

        # FakeActManager
        RET_NST = {"resp": "NotSupport"}
        def doAction(self, action, session):
            ret = RET_NST
            actFun = self.__class__.__dict__.get("action_%s" % action, None)
            if actFun:.
                user = session.get("user", User())
                unscoped_token_id = session.get("unscoped_token", "")
                self.request = Request(user, unscoped_token_id)
                ret = actFun(self)
            return ret

        # RespMgmt
        def addResponse(self, reqId, resp):
            self.respDict[reqId] = resp
            return True

总结一下各个模块的作用：

* ***Dispacher***  
    异步机制接口  
    `sendRequest()`：把任务发给后台处理，后台自动保存任务返回数据  
    `getResponse()`：用ID找回任务返回数据  

* ***SessionMgmt***  
    维护session及有效时间  

* ***ActionProxy***  
    代理；从request取出Action交给FakeActManager处理

* ***ActionMgmt***  
    空的（未完成）

* ***RespMgmt***  
    管理任务返回的数据（通过ID从字典里查找）

* ***FakeActManager***  
    执行任务，并将返回结果转换数据格式


{% endraw %}

