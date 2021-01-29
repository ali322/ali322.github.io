---
layout: post
title: Cnoder 迁移记
date: 2019-02-10 11:13:30
tags: Flutter
---

## 前言

之前在学会 React-Native 后写了一个 cnodejs社区的客户端 [CNodeRN](https://github.com/ali322/CNodeRN),前阵子了解了下 flutter, 感觉是移动应用开发的未来趋势,便有了迁移至 flutter 技术栈的想法, 然后就有了 [CNoder](https://github.com/ali322/cnoder) 这个项目, 也算是对数周 flutter 的一个学习实践吧

<!-- more -->

## 安装和初始化

跟着官方的[安装说明](https://flutter.io/setup-macos/)一步一步往下走,还是挺顺利的,唯一不同的就是增加了镜像设置这一步, 打开 `~/.zhsrc`, 末尾增加

```bash
 ## flutter
125 export PUB_HOSTED_URL=https://pub.flutter-io.cn
126 export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
127 export PATH=$HOME/flutter/bin:$PATH
```
然后执行 `flutter doctor` 检查环境是否正常,一切顺利的话就可以初始化项目了,我使用的编辑器是 `vscode`, 通过命令窗口运行命令 `Flutter: New Project` 即可

## 项目目录结构

源码都位于 `lib` 目录下

```bash
|-- config/
    |-- api.dart // http api 调用接口地址配置
|-- common/
    |-- helper.dart // 工具函数
|-- route/
    |-- handler.dart // 路由配置文件
|-- store/
    |-- action/  // redux action 目录
    |-- epic/   // redux_epic 配置目录
    |-- reducer/ // redux reducer 目录
    |-- model/ // 模型目录
    |-- view_model/ // store 映射模型目录
    |-- root_state.dart // 全局 state
    |-- index.dart // store 初始入口
|-- container/  // 连接 store 的容器目录
|-- widget/ // 视图 widget 目录
main.dart // 入口文件
app.dart // 入口widget
```

## 功能模块

- 入口文件: main.dart, 逻辑很简单就不描述了
- 入口widget: app.dart文件

```dart
class App extends StatelessWidget {
  // 初始化路由插件
  final Router router = new Router();

  App() {
    // 从持久化存储里加载数据状态,这里用来存储用户的身份令牌信息
    persistor.load(store);
    // 404处理
    router.notFoundHandler = notFoundHandler;
    // 应用路由配置
    handlers.forEach((String path,Handler handler) {
      router.define(path, handler: handler);
    });
  }

  @override
    Widget build(BuildContext context) {
      final app = new MaterialApp(
        title: 'CNoder',
        // 禁用右上角的 debug 标志
        debugShowCheckedModeBanner: false,
        theme: new ThemeData(
          primarySwatch: Colors.lightGreen,
          // 定义全局图标主题
          iconTheme: new IconThemeData(
            color: Color(0xFF666666)
          ),
          // 定义全局文本主题
          textTheme: new TextTheme(
            body1: new TextStyle(color: Color(0xFF333333), fontSize: 14.0)
          )
        ),
        // 将 应用的路由映射至 fluro 的路由表里面去
        onGenerateRoute: router.generator
      );

      return  new StoreProvider<RootState>(store: store, child: app);
    }
}
```

这里有个坑,如果按照 fluro 提供的文档将应用路由映射至fluro的路由表,使用的方式是 `onGenerateRoute: router.generator`, 但是这样的话在路由跳转时就无法指定过渡动效了,因此需要改成这样

```dart
onGenerateRoute: (RouteSettings routeSettings) {
  // 这个方法可以在 router.generator 源码里找到,返回匹配的路由
  RouteMatch match = this.router.matchRoute(null, routeSettings.name, routeSettings: routeSettings, transitionType: TransitionType.inFromRight);
  return match.route;
},
```

使用 StoreProvider 容器包裹整个应用入口widget,这样才能在子节点的widget上使用StoreConnector连接store来获取数据状态和派发action

- 接下来应用会进入路由机制,下面是部分路由配置信息

```dart
import "dart:core";
import "package:fluro/fluro.dart";
import "package:flutter/material.dart";
import "package:cnoder/container/index.dart";

Map<String, Handler> handlers = {
  '/': new Handler(
      handlerFunc: (BuildContext context, Map<String, dynamic> params) {
    return new IndexContainer();
  }),
  ...
};
```

`container/index.dart` 类似于 react 里面的 HOC,将 store 连接至子widget

```dart
import "package:flutter/material.dart";
import "package:redux/redux.dart";
import "package:flutter_redux/flutter_redux.dart";
import "../store/root_state.dart";
import "../store/view_model/index.dart";
import "../widget/index.dart";

class IndexContainer extends StatelessWidget{
  @override
    Widget build(BuildContext context) {
      return new StoreConnector<RootState, IndexViewModel>(
        converter: (Store<RootState> store) => IndexViewModel.fromStore(store),
        builder: (BuildContext context, IndexViewModel vm) {
          return new IndexScene(vm: vm);
        },
      );
    }
}
```

converter 参数相当于在使用 react+redux 技术栈里面的使用 connect 函数包裹组件时的 mapAction 和 mapState 参数,将返回值作为 builder 参数对应的回调函数第二个入参 vm.

- `widget/index.dart` 为首页的视图widget,通过底部的标签栏切换四个容器widget的显示

```dart
class IndexState extends State<IndexScene> {
  // 根据登陆状态切换显示
  List _renderScenes(bool isLogined) {
    final bool isLogined = widget.vm.auth["isLogined"];
    return <Widget>[
      new TopicsContainer(vm: widget.vm),
      isLogined ? new CollectContainer(vm: widget.vm) : new LoginScene(),
      isLogined ? new MessageContainer(vm: widget.vm,) : new LoginScene(),
      isLogined ? new MeContainer(vm: widget.vm,) : new LoginScene()
    ];
  }

  @override
    Widget build(BuildContext context) {
      final bool isLogined = widget.vm.auth["isLogined"];
      final List scenes = _renderScenes(isLogined);
      final int tabIndex = widget.vm.tabIndex;
      final Function setTab = widget.vm.selectTab;
      
      final currentScene = scenes[0];
      // 这里保证了初始化widget的服务调用
      if (currentScene is InitializeContainer) {
        if (currentScene.getInitialized() == false) {
          currentScene.initialize();
          currentScene.setInitialized();
        }
      }

      return new Scaffold(
        bottomNavigationBar: new CupertinoTabBar(
          activeColor: Colors.green,
          backgroundColor: const Color(0xFFF7F7F7),
          currentIndex: tabIndex,
          onTap: (int i) {
            final currentScene = scenes[i];
            if (isLogined) {
              //  这里保证了widget的服务调用在切换时只进行一次
              if (currentScene is InitializeContainer) {
                if (currentScene.getInitialized() == false) {
                  currentScene.initialize();
                  currentScene.setInitialized();
                }
              }
            }
            setTab(i);
          },
          items: <BottomNavigationBarItem>[
            new BottomNavigationBarItem(
              icon: new Icon(Icons.home),
              title: new Text('主题'),
            ),
            new BottomNavigationBarItem(
              icon: new Icon(Icons.favorite),
              title: new Text('收藏')
            ),
            new BottomNavigationBarItem(
              icon: new Icon(Icons.message),
              title: new Text('消息')
            ),
            new BottomNavigationBarItem(
              icon: new Icon(Icons.person),
              title: new Text('我的')
            )
          ],
        ),
        // 使用层叠widget来包裹视图,同一时间仅一个视图widget可见
        body: new IndexedStack(
          children: scenes,
          index: tabIndex,
        )
      );
    }
}
```

很多同学会有疑问,tabIndex 这个应该只是首页widget的内部数据状态,为何要放到 redux 里去维护?因为我们在子widget里面会去切换页签的选中状态,比如登陆完成以后切换至'我的'这个页签

- 主题视图容器widget,在容器组件里面触发服务调用获取主题数据

```dart
// 初始化标志位
bool initialized = false;

class TopicsContainer extends StatelessWidget implements InitializeContainer{
  final IndexViewModel vm;

  TopicsContainer({Key key, @required this.vm}):super(key: key);

  // 标记已初始化,防止在首页页签切换时重复调用
  void setInitialized() {
    initialized = true;
  }

  // 获取初始化状态
  bool getInitialized() {
    return initialized;
  }

  // 初始化的操作是调用 redux action 获取主题数据
  void initialize() {
    vm.fetchTopics();
  }

  @override
    Widget build(BuildContext context) {
      return new StoreConnector<RootState, TopicsViewModel>(
        converter: (Store<RootState> store) => TopicsViewModel.fromStore(store),
        builder: (BuildContext context, TopicsViewModel vm) {
          return new TopicsScene(vm: vm);
        },
      );
    }
}
```

- 主题视图widget,顶部四个页签用来切换显示四个主题分类

```dart
class TopicsState extends State<TopicsScene> with TickerProviderStateMixin{
  @override
  void initState() {
    super.initState();
    final topicsOfCategory = widget.vm.topicsOfCategory;

    _tabs = <Tab>[];
    // 初始化顶部页签栏
    topicsOfCategory.forEach((k, v) {
      _tabs.add(new Tab(
        text: v["label"]
      ));
    });
    // 初始化 TabBar 和 TabBarView 的控制器
    _tabController  = new TabController(
      length: _tabs.length,
      vsync: this // _tabController 作为属性的类必须通过 TickerProviderStateMixin 扩展
    );
    
    // 页签切换事件监听
    _onTabChange = () {
        ...
    };
    
    // 给页签控制器增加一个事件监听器,监听页签切换事件
    _tabController.addListener(_onTabChange);
  }
  
  @override
  void dispose() {
    super.dispose();
    // 类销毁之前移除页签控制器的事件监听
    _tabController.removeListener(_onTabChange);
    // 销毁页签控制器
    _tabController.dispose();
  }
  
  @override
    Widget build(BuildContext context) {
      bool isLoading = widget.vm.isLoading;
      Map topicsOfCategory = widget.vm.topicsOfCategory;
      FetchTopics fetchTopics = widget.vm.fetchTopics;
      ResetTopics resetTopics = widget.vm.resetTopics;
      
      ...
      
      // 循环显示分类下的主题列表
      List<Widget> _renderTabView() {
        final _tabViews = <Widget>[];
        topicsOfCategory.forEach((k, category) {
          bool isFetched = topicsOfCategory[k]["isFetched"];
          // 如果该分类下的主题列表未初始化先渲染一个加载指示
          _tabViews.add(!isFetched ? _renderLoading(context) :
          // 使用 pull_to_refresh 包提供的下拉刷新和上来加载功能
          new SmartRefresher(
            enablePullDown: true,
            enablePullUp: true,
            onRefresh: _onRefresh(k),
            controller: _controller,
            child: new ListView.builder(
              physics: const NeverScrollableScrollPhysics(),
              shrinkWrap: true,
              itemCount: topicsOfCategory[k]["list"].length,
              itemBuilder: (BuildContext context, int i) => _renderRow(context, topicsOfCategory[k]["list"][i]),
            ),
          ));
        });
        return _tabViews;
      }
      
      // 使用 ListTile 渲染列表中的每一行
      Widget _renderRow(BuildContext context, Topic topic) {
      ListTile title = new ListTile(
        leading: new SizedBox(
          width: 30.0,
          height: 30.0,
          // 使用 cached_network_image 提供支持缓存和占位图的功能显示头像
          child: new CachedNetworkImage(
            imageUrl: topic.authorAvatar.startsWith('//') ? 'http:${topic.authorAvatar}' : topic.authorAvatar,
            placeholder: new Image.asset('asset/image/cnoder_avatar.png'),
            errorWidget: new Icon(Icons.error),
          )
        ),
        title: new Text(topic.authorName),
        subtitle: new Row(
          children: <Widget>[
            new Text(topic.lastReplyAt)
          ],
        ),
        trailing: new Text('${topic.replyCount}/${topic.visitCount}'),
      );
      return new InkWell(
        // 点击后跳转至主题详情
        onTap: () => Navigator.of(context).pushNamed('/topic/${topic.id}'),
        child: new Column(
          children: <Widget>[
            title,
            new Container(
              padding: const EdgeInsets.all(10.0),
              alignment: Alignment.centerLeft,
              child: new Text(topic.title),
            )
          ],
        ),
      );
    }
    
      return new Scaffold(
        appBar: new AppBar(
          brightness: Brightness.dark,
          elevation: 0.0,
          titleSpacing: 0.0,
          bottom: null,
          // 顶部显示页签栏
          title: new Align(
            alignment: Alignment.bottomCenter,
            child: new TabBar(
              labelColor: Colors.white,
              tabs: _tabs,
              controller: _tabController,
            )
          )
        ),
        // 主体区域显示页签内容
        body: new TabBarView(
          controller: _tabController,
          children: _renderTabView(),
        )
      );
    }
}
```

## 数据状态

- `store/view_model/topics.dart` 视图映射模型定义

通过视图映射模型将 store 里面的 state 和 action 传递给视图widget,
在上面的主题容器widget里面我们通过 `vm.fetchTopics` 方法获取主题数据, 这个方法是在 TopicsViewModel 这个
store 映射模型里定义的

```dart
class TopicsViewModel {
  final Map topicsOfCategory;
  final bool isLoading;
  final FetchTopics fetchTopics;
  final ResetTopics resetTopics;

  TopicsViewModel({
    @required this.topicsOfCategory, 
    @required this.isLoading, 
    @required this.fetchTopics, 
    @required this.resetTopics
  });

  static TopicsViewModel fromStore(Store<RootState> store) {
    return new TopicsViewModel(
      // 映射分类主题列表
      topicsOfCategory: store.state.topicsOfCategory,
      // 映射加载状态
      isLoading: store.state.isLoading,
      // 获取主题数据 action 的包装方法
      fetchTopics: ({int currentPage = 1, String category = '', Function afterFetched = _noop}) {
        // 通过 isLoading 数据状态的变更来切换widget的加载指示器的显示
        store.dispatch(new ToggleLoading(true));
        // 触发获取主题数据的action,将当前页,分类名,以及调用成功的回调函数传递给action
        store.dispatch(new RequestTopics(currentPage: currentPage, category: category, afterFetched: afterFetched));
      },
      // 刷新主题数据的包装方法
      resetTopics: ({@required String category, @required Function afterFetched}) {
        store.dispatch(new RequestTopics(currentPage: 1, category: category, afterFetched: afterFetched));
      }
    );
  }
}
```

这里增加了一个调用成功的回调函数给 action,是因为需要在 http 服务调用完成以后控制主题视图widget里面 SmartRefresher 这个widget 状态的切换(重置加载指示等等)

```dart
final _onRefresh = (String category) {
    return (bool up) {
      // 如果是上拉加载更多
      if (!up) {
        if (isLoading) {
          _controller.sendBack(false, RefreshStatus.idle);
          return;
        }
        fetchTopics(
          currentPage: topicsOfCategory[category]["currentPage"] + 1,
          category: category,
          afterFetched: () {
            // 上拉加载更多指示器复位
            _controller.sendBack(false, RefreshStatus.idle);
          }
        );
      // 如果是下拉刷新
      } else {
        resetTopics(
          category: category,
          afterFetched: () {
            // 下拉刷新指示器复位
            _controller.sendBack(true, RefreshStatus.completed);
          }
        );
      }
    };
  };
```

- `store/action/topic.dart` action 定义

在 flutter 中以类的方式来定义 action 的,这一点与我们在 react 中使用 redux 有点不同

```dart
// 发送主题列表请求的 action
class RequestTopics {
  // 当前页
  final int currentPage;
  // 分类
  final String category;
  // 请求完成的回调
  final VoidCallback afterFetched;

  RequestTopics({this.currentPage = 1, this.category = "", @required this.afterFetched});
}

// 响应主题列表请求的 action
class ResponseTopics {
  final List<Topic> topics;
  final int currentPage;
  final String category;

  ResponseTopics(this.currentPage, this.category, this.topics);

  ResponseTopics.failed() : this(1, "", []);
}
```

- epic 定义,redux epic 可以看成是 action 的一个调度器,虽然 flutter 里的redux 也有 redux_thunk 中间件,但是 epic 这种基于流的调度中间件使得业务逻辑更加优雅

```dart
Stream<dynamic> fetchTopicsEpic(
    Stream<dynamic> actions, EpicStore<RootState> store) {
  return new Observable(actions)
      // 过滤特定请求
      .ofType(new TypeToken<RequestTopics>())
      .flatMap((action) {
        // 通过异步生成器来构建一个流
        return new Observable(() async* {
          try {
            // 发送获取主题列表的 http 请求
            final ret = await http.get("${apis['topics']}?page=${action.currentPage}&limit=6&tab=${action.category}&mdrender=false");
            Map<String, dynamic> result = json.decode(ret.body);
            List<Topic> topics = [];
            result['data'].forEach((v) {
              topics.add(new Topic.fromJson(v));
            });
            // 触发请求完成的回调,就是我们上面提到的 SmartRefresher widget 的复位
            action.afterFetched();
            yield new ResponseTopics(action.currentPage, action.category, topics);
          } catch(err) {
            print(err);
            yield new ResponseTopicsFailed(err);
          }
          // 刷新数据状态复位
          yield new ToggleLoading(false);
        } ());
  });
}
```

在接收到请求响应后,通过 `Topic.fromJson` 这个指定类构造器来创建主题列表,这个方法定义在 `store/model/topic.dart`里面

```dart
Topic.fromJson(final Map map):
    this.id = map["id"],
    this.authorName = map["author"]["loginname"],
    this.authorAvatar = map["author"]["avatar_url"],
    this.title = map["title"],
    this.tag = map["tab"],
    this.content = map["content"],
    this.createdAt = fromNow(map["create_at"]),
    this.lastReplyAt = fromNow(map["last_reply_at"]),
    this.replyCount = map["reply_count"],
    this.visitCount = map["visit_count"],
    this.top = map["top"],
    this.isCollect = map["is_collect"],
    this.replies = formatedReplies(map['replies']);
```

- `store/reducer/topic.dart`, 通过主题列表的 reducer 来变更 store 里面的数据状态

```dart
final Reducer<Map> topicsReducer = combineReducers([
  // 通过指定 action 类型来拆分
  new TypedReducer<Map, ClearTopic>(_clearTopic),
  new TypedReducer<Map, RequestTopics>(_requestTopics),
  new TypedReducer<Map, ResponseTopics>(_responseTopics)
]);

// 清空主题列表
Map _clearTopic(Map state, ClearTopic action) {
  return {};
}

Map _requestTopics(Map state, RequestTopics action) {
  Map topicsOfTopics = {};
  state.forEach((k, v) {
    final _v = new Map.from(v);
    if (action.category == k) {
      // 通过 isFetched 标志位来防止分类页面切换时重复请求
      _v["isFetched"] = false;
    }
    topicsOfTopics[k] = _v;
  });
  return topicsOfTopics;
}

Map _responseTopics(Map state, ResponseTopics action) {
  Map topicsOfCategory = {};
  state.forEach((k, v) {
    Map _v = {};
    _v.addAll(v);
    if (k == action.category) {
      List _list = [];
      // 上拉加载更多时
      if (_v['currentPage'] < action.currentPage) {
        _list.addAll(_v["list"]);
        _list.addAll(action.topics);
      } 
      // 下拉刷新时
      if (action.currentPage == 1) {
        _list.addAll(action.topics);
      }
      // 通过 isFetched 标志位来防止分类页面切换时重复请求
      _v["isFetched"] = true;
      _v["list"] = _list;
      _v["currentPage"] = action.currentPage;
    }
    topicsOfCategory[k] = _v;
  });
  return topicsOfCategory;
}
```

然后在 `store/reducer/root.dart` 的 rootReducer 里进行合并

```dart
RootState rootReducer(RootState state, action) {
  // 处理从持久化存储里加载数据状态
  if (action is PersistLoadedAction<RootState>) {
    return action.state ?? state;
  }
  // 将 state 里的数据状态对应到子 reducer
  return new RootState(
    tabIndex: tabReducer(state.tabIndex, action),
    auth:  loginReducer(state.auth, action),
    isLoading: loadingReducer(state.isLoading, action),
    topicsOfCategory: topicsReducer(state.topicsOfCategory, action),
    topic: topicReducer(state.topic, action),
    me: meReducer(state.me, action),
    collects: collectsReducer(state.collects, action),
    messages: messagesReducer(state.messages, action)
  );
}
```

- `store/index.dart` store 的初始化入口,在我们上面的入口widget里面使用 `StoreProvider` 容器包裹的时候传递

```dart
// 合并 epic 获得根 epic 提供给 epic 中间件调用
final epic = combineEpics([
  doLoginEpic, 
  fetchTopicsEpic, fetchTopicEpic, 
  fetchMeEpic,
  fetchCollectsEpic,
  fetchMessagesEpic,
  fetchMessageCountEpic,
  markAllAsReadEpic,
  markAsReadEpic,
  createReplyEpic,
  saveTopicEpic,
  createTopicEpic,
  toggleCollectEpic,
  likeReplyEpic,
]);

// 初始化持久化中间件存储容器
final persistor = Persistor<RootState>(
  storage: FlutterStorage('cnoder'),
  decoder: RootState.fromJson,
  debug: true
);

// 初始化 store
final store = new Store<RootState>(rootReducer,
  initialState: new RootState(), middleware: [
    new LoggingMiddleware.printer(), 
    new EpicMiddleware(epic),
    persistor.createMiddleware()
]);
```

这里有个小坑,持久化存储中间件 redux_persist 的文档上加载中间件的方式为

```dart
var store = new Store<AppState>(
  reducer,
  initialState: new AppState(),
  middleware: [persistor.createMiddleware()],
);
```

但是这样处理的话,在每个业务 action 触发的时候,都会触发持久化的操作,而这在很多场景下是不必要的,比如在我们的应用中只需要保存的用户身份令牌,所以只需要在触发登陆和登出 action 的时候执行持久化的操作,因此加载中间件的方式需要做如下改动

```dart
void persistMiddleware(Store store, dynamic action, NextDispatcher next) {
  next(action);
  // 仅处理登陆和登出操作
  if (action is FinishLogin || action is Logout) {
    try {
      persistor.save(store);
    } catch (_) {}
  }
}

// 初始化 store
final store = new Store<RootState>(rootReducer,
  initialState: new RootState(), middleware: [
    new LoggingMiddleware.printer(), 
    new EpicMiddleware(epic),
    persistMiddleware
]);
```

## 更多

应用的视图层和数据状态处理还是跟使用 React-Native 开发中使用 redux 技术栈的方式差不多,虽然整体目录结构有点繁琐,但是业务逻辑清晰明了,在后续功能扩展和维护的时候还是带来不少的方便,唯一遗憾的是因为 flutter 系统架构的问题,还没有一个针对 flutter 的 redux devtools,这一点还是蛮影响开发效率的

完整的项目源码请关注github仓库: [cnoder](https://github.com/ali322/cnoder),欢迎 star 和 PR,对 flutter 理解的不深,还望各位对本文中的不足之处批评指正

**![从 0 到 1：我的 Flutter 技术实践 | 掘金技术征文，征文活动整在进行中](https://juejin.im/post/5b43187ff265da0f491b87e7)**









## 前言

之前在学会 React-Native 后写了一个 cnodejs社区的客户端 [CNodeRN](https://github.com/ali322/CNodeRN),前阵子了解了下 flutter, 感觉是移动应用开发的未来趋势,便有了迁移至 flutter 技术栈的想法, 然后就有了 [CNoder](https://github.com/ali322/cnoder) 这个项目, 也算是对数周 flutter 的一个学习实践吧

## 安装和初始化

跟着官方的[安装说明](https://flutter.io/setup-macos/)一步一步往下走,还是挺顺利的,唯一不同的就是增加了镜像设置这一步, 打开 `~/.zhsrc`, 末尾增加

```bash
 ## flutter
125 export PUB_HOSTED_URL=https://pub.flutter-io.cn
126 export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
127 export PATH=$HOME/flutter/bin:$PATH
```
然后执行 `flutter doctor` 检查环境是否正常,一切顺利的话就可以初始化项目了,我使用的编辑器是 `vscode`, 通过命令窗口运行命令 `Flutter: New Project` 即可

## 项目目录结构

源码都位于 `lib` 目录下

```bash
|-- config/
    |-- api.dart // http api 调用接口地址配置
|-- common/
    |-- helper.dart // 工具函数
|-- route/
    |-- handler.dart // 路由配置文件
|-- store/
    |-- action/  // redux action 目录
    |-- epic/   // redux_epic 配置目录
    |-- reducer/ // redux reducer 目录
    |-- model/ // 模型目录
    |-- view_model/ // store 映射模型目录
    |-- root_state.dart // 全局 state
    |-- index.dart // store 初始入口
|-- container/  // 连接 store 的容器目录
|-- widget/ // 视图 widget 目录
main.dart // 入口文件
app.dart // 入口widget
```

## 功能模块

- 入口文件: main.dart, 逻辑很简单就不描述了
- 入口widget: app.dart文件

```dart
class App extends StatelessWidget {
  // 初始化路由插件
  final Router router = new Router();

  App() {
    // 从持久化存储里加载数据状态,这里用来存储用户的身份令牌信息
    persistor.load(store);
    // 404处理
    router.notFoundHandler = notFoundHandler;
    // 应用路由配置
    handlers.forEach((String path,Handler handler) {
      router.define(path, handler: handler);
    });
  }

  @override
    Widget build(BuildContext context) {
      final app = new MaterialApp(
        title: 'CNoder',
        // 禁用右上角的 debug 标志
        debugShowCheckedModeBanner: false,
        theme: new ThemeData(
          primarySwatch: Colors.lightGreen,
          // 定义全局图标主题
          iconTheme: new IconThemeData(
            color: Color(0xFF666666)
          ),
          // 定义全局文本主题
          textTheme: new TextTheme(
            body1: new TextStyle(color: Color(0xFF333333), fontSize: 14.0)
          )
        ),
        // 将 应用的路由映射至 fluro 的路由表里面去
        onGenerateRoute: router.generator
      );

      return  new StoreProvider<RootState>(store: store, child: app);
    }
}
```

这里有个坑,如果按照 fluro 提供的文档将应用路由映射至fluro的路由表,使用的方式是 `onGenerateRoute: router.generator`, 但是这样的话在路由跳转时就无法指定过渡动效了,因此需要改成这样

```dart
onGenerateRoute: (RouteSettings routeSettings) {
  // 这个方法可以在 router.generator 源码里找到,返回匹配的路由
  RouteMatch match = this.router.matchRoute(null, routeSettings.name, routeSettings: routeSettings, transitionType: TransitionType.inFromRight);
  return match.route;
},
```

使用 StoreProvider 容器包裹整个应用入口widget,这样才能在子节点的widget上使用StoreConnector连接store来获取数据状态和派发action

- 接下来应用会进入路由机制,下面是部分路由配置信息

```dart
import "dart:core";
import "package:fluro/fluro.dart";
import "package:flutter/material.dart";
import "package:cnoder/container/index.dart";

Map<String, Handler> handlers = {
  '/': new Handler(
      handlerFunc: (BuildContext context, Map<String, dynamic> params) {
    return new IndexContainer();
  }),
  ...
};
```

`container/index.dart` 类似于 react 里面的 HOC,将 store 连接至子widget

```dart
import "package:flutter/material.dart";
import "package:redux/redux.dart";
import "package:flutter_redux/flutter_redux.dart";
import "../store/root_state.dart";
import "../store/view_model/index.dart";
import "../widget/index.dart";

class IndexContainer extends StatelessWidget{
  @override
    Widget build(BuildContext context) {
      return new StoreConnector<RootState, IndexViewModel>(
        converter: (Store<RootState> store) => IndexViewModel.fromStore(store),
        builder: (BuildContext context, IndexViewModel vm) {
          return new IndexScene(vm: vm);
        },
      );
    }
}
```

converter 参数相当于在使用 react+redux 技术栈里面的使用 connect 函数包裹组件时的 mapAction 和 mapState 参数,将返回值作为 builder 参数对应的回调函数第二个入参 vm.

- `widget/index.dart` 为首页的视图widget,通过底部的标签栏切换四个容器widget的显示

```dart
class IndexState extends State<IndexScene> {
  // 根据登陆状态切换显示
  List _renderScenes(bool isLogined) {
    final bool isLogined = widget.vm.auth["isLogined"];
    return <Widget>[
      new TopicsContainer(vm: widget.vm),
      isLogined ? new CollectContainer(vm: widget.vm) : new LoginScene(),
      isLogined ? new MessageContainer(vm: widget.vm,) : new LoginScene(),
      isLogined ? new MeContainer(vm: widget.vm,) : new LoginScene()
    ];
  }

  @override
    Widget build(BuildContext context) {
      final bool isLogined = widget.vm.auth["isLogined"];
      final List scenes = _renderScenes(isLogined);
      final int tabIndex = widget.vm.tabIndex;
      final Function setTab = widget.vm.selectTab;
      
      final currentScene = scenes[0];
      // 这里保证了初始化widget的服务调用
      if (currentScene is InitializeContainer) {
        if (currentScene.getInitialized() == false) {
          currentScene.initialize();
          currentScene.setInitialized();
        }
      }

      return new Scaffold(
        bottomNavigationBar: new CupertinoTabBar(
          activeColor: Colors.green,
          backgroundColor: const Color(0xFFF7F7F7),
          currentIndex: tabIndex,
          onTap: (int i) {
            final currentScene = scenes[i];
            if (isLogined) {
              //  这里保证了widget的服务调用在切换时只进行一次
              if (currentScene is InitializeContainer) {
                if (currentScene.getInitialized() == false) {
                  currentScene.initialize();
                  currentScene.setInitialized();
                }
              }
            }
            setTab(i);
          },
          items: <BottomNavigationBarItem>[
            new BottomNavigationBarItem(
              icon: new Icon(Icons.home),
              title: new Text('主题'),
            ),
            new BottomNavigationBarItem(
              icon: new Icon(Icons.favorite),
              title: new Text('收藏')
            ),
            new BottomNavigationBarItem(
              icon: new Icon(Icons.message),
              title: new Text('消息')
            ),
            new BottomNavigationBarItem(
              icon: new Icon(Icons.person),
              title: new Text('我的')
            )
          ],
        ),
        // 使用层叠widget来包裹视图,同一时间仅一个视图widget可见
        body: new IndexedStack(
          children: scenes,
          index: tabIndex,
        )
      );
    }
}
```

很多同学会有疑问,tabIndex 这个应该只是首页widget的内部数据状态,为何要放到 redux 里去维护?因为我们在子widget里面会去切换页签的选中状态,比如登陆完成以后切换至'我的'这个页签

- 主题视图容器widget,在容器组件里面触发服务调用获取主题数据

```dart
// 初始化标志位
bool initialized = false;

class TopicsContainer extends StatelessWidget implements InitializeContainer{
  final IndexViewModel vm;

  TopicsContainer({Key key, @required this.vm}):super(key: key);

  // 标记已初始化,防止在首页页签切换时重复调用
  void setInitialized() {
    initialized = true;
  }

  // 获取初始化状态
  bool getInitialized() {
    return initialized;
  }

  // 初始化的操作是调用 redux action 获取主题数据
  void initialize() {
    vm.fetchTopics();
  }

  @override
    Widget build(BuildContext context) {
      return new StoreConnector<RootState, TopicsViewModel>(
        converter: (Store<RootState> store) => TopicsViewModel.fromStore(store),
        builder: (BuildContext context, TopicsViewModel vm) {
          return new TopicsScene(vm: vm);
        },
      );
    }
}
```

- 主题视图widget,顶部四个页签用来切换显示四个主题分类

```dart
class TopicsState extends State<TopicsScene> with TickerProviderStateMixin{
  @override
  void initState() {
    super.initState();
    final topicsOfCategory = widget.vm.topicsOfCategory;

    _tabs = <Tab>[];
    // 初始化顶部页签栏
    topicsOfCategory.forEach((k, v) {
      _tabs.add(new Tab(
        text: v["label"]
      ));
    });
    // 初始化 TabBar 和 TabBarView 的控制器
    _tabController  = new TabController(
      length: _tabs.length,
      vsync: this // _tabController 作为属性的类必须通过 TickerProviderStateMixin 扩展
    );
    
    // 页签切换事件监听
    _onTabChange = () {
        ...
    };
    
    // 给页签控制器增加一个事件监听器,监听页签切换事件
    _tabController.addListener(_onTabChange);
  }
  
  @override
  void dispose() {
    super.dispose();
    // 类销毁之前移除页签控制器的事件监听
    _tabController.removeListener(_onTabChange);
    // 销毁页签控制器
    _tabController.dispose();
  }
  
  @override
    Widget build(BuildContext context) {
      bool isLoading = widget.vm.isLoading;
      Map topicsOfCategory = widget.vm.topicsOfCategory;
      FetchTopics fetchTopics = widget.vm.fetchTopics;
      ResetTopics resetTopics = widget.vm.resetTopics;
      
      ...
      
      // 循环显示分类下的主题列表
      List<Widget> _renderTabView() {
        final _tabViews = <Widget>[];
        topicsOfCategory.forEach((k, category) {
          bool isFetched = topicsOfCategory[k]["isFetched"];
          // 如果该分类下的主题列表未初始化先渲染一个加载指示
          _tabViews.add(!isFetched ? _renderLoading(context) :
          // 使用 pull_to_refresh 包提供的下拉刷新和上来加载功能
          new SmartRefresher(
            enablePullDown: true,
            enablePullUp: true,
            onRefresh: _onRefresh(k),
            controller: _controller,
            child: new ListView.builder(
              physics: const NeverScrollableScrollPhysics(),
              shrinkWrap: true,
              itemCount: topicsOfCategory[k]["list"].length,
              itemBuilder: (BuildContext context, int i) => _renderRow(context, topicsOfCategory[k]["list"][i]),
            ),
          ));
        });
        return _tabViews;
      }
      
      // 使用 ListTile 渲染列表中的每一行
      Widget _renderRow(BuildContext context, Topic topic) {
      ListTile title = new ListTile(
        leading: new SizedBox(
          width: 30.0,
          height: 30.0,
          // 使用 cached_network_image 提供支持缓存和占位图的功能显示头像
          child: new CachedNetworkImage(
            imageUrl: topic.authorAvatar.startsWith('//') ? 'http:${topic.authorAvatar}' : topic.authorAvatar,
            placeholder: new Image.asset('asset/image/cnoder_avatar.png'),
            errorWidget: new Icon(Icons.error),
          )
        ),
        title: new Text(topic.authorName),
        subtitle: new Row(
          children: <Widget>[
            new Text(topic.lastReplyAt)
          ],
        ),
        trailing: new Text('${topic.replyCount}/${topic.visitCount}'),
      );
      return new InkWell(
        // 点击后跳转至主题详情
        onTap: () => Navigator.of(context).pushNamed('/topic/${topic.id}'),
        child: new Column(
          children: <Widget>[
            title,
            new Container(
              padding: const EdgeInsets.all(10.0),
              alignment: Alignment.centerLeft,
              child: new Text(topic.title),
            )
          ],
        ),
      );
    }
    
      return new Scaffold(
        appBar: new AppBar(
          brightness: Brightness.dark,
          elevation: 0.0,
          titleSpacing: 0.0,
          bottom: null,
          // 顶部显示页签栏
          title: new Align(
            alignment: Alignment.bottomCenter,
            child: new TabBar(
              labelColor: Colors.white,
              tabs: _tabs,
              controller: _tabController,
            )
          )
        ),
        // 主体区域显示页签内容
        body: new TabBarView(
          controller: _tabController,
          children: _renderTabView(),
        )
      );
    }
}
```

## 数据状态

- `store/view_model/topics.dart` 视图映射模型定义

通过视图映射模型将 store 里面的 state 和 action 传递给视图widget,
在上面的主题容器widget里面我们通过 `vm.fetchTopics` 方法获取主题数据, 这个方法是在 TopicsViewModel 这个
store 映射模型里定义的

```dart
class TopicsViewModel {
  final Map topicsOfCategory;
  final bool isLoading;
  final FetchTopics fetchTopics;
  final ResetTopics resetTopics;

  TopicsViewModel({
    @required this.topicsOfCategory, 
    @required this.isLoading, 
    @required this.fetchTopics, 
    @required this.resetTopics
  });

  static TopicsViewModel fromStore(Store<RootState> store) {
    return new TopicsViewModel(
      // 映射分类主题列表
      topicsOfCategory: store.state.topicsOfCategory,
      // 映射加载状态
      isLoading: store.state.isLoading,
      // 获取主题数据 action 的包装方法
      fetchTopics: ({int currentPage = 1, String category = '', Function afterFetched = _noop}) {
        // 通过 isLoading 数据状态的变更来切换widget的加载指示器的显示
        store.dispatch(new ToggleLoading(true));
        // 触发获取主题数据的action,将当前页,分类名,以及调用成功的回调函数传递给action
        store.dispatch(new RequestTopics(currentPage: currentPage, category: category, afterFetched: afterFetched));
      },
      // 刷新主题数据的包装方法
      resetTopics: ({@required String category, @required Function afterFetched}) {
        store.dispatch(new RequestTopics(currentPage: 1, category: category, afterFetched: afterFetched));
      }
    );
  }
}
```

这里增加了一个调用成功的回调函数给 action,是因为需要在 http 服务调用完成以后控制主题视图widget里面 SmartRefresher 这个widget 状态的切换(重置加载指示等等)

```dart
final _onRefresh = (String category) {
    return (bool up) {
      // 如果是上拉加载更多
      if (!up) {
        if (isLoading) {
          _controller.sendBack(false, RefreshStatus.idle);
          return;
        }
        fetchTopics(
          currentPage: topicsOfCategory[category]["currentPage"] + 1,
          category: category,
          afterFetched: () {
            // 上拉加载更多指示器复位
            _controller.sendBack(false, RefreshStatus.idle);
          }
        );
      // 如果是下拉刷新
      } else {
        resetTopics(
          category: category,
          afterFetched: () {
            // 下拉刷新指示器复位
            _controller.sendBack(true, RefreshStatus.completed);
          }
        );
      }
    };
  };
```

- `store/action/topic.dart` action 定义

在 flutter 中以类的方式来定义 action 的,这一点与我们在 react 中使用 redux 有点不同

```dart
// 发送主题列表请求的 action
class RequestTopics {
  // 当前页
  final int currentPage;
  // 分类
  final String category;
  // 请求完成的回调
  final VoidCallback afterFetched;

  RequestTopics({this.currentPage = 1, this.category = "", @required this.afterFetched});
}

// 响应主题列表请求的 action
class ResponseTopics {
  final List<Topic> topics;
  final int currentPage;
  final String category;

  ResponseTopics(this.currentPage, this.category, this.topics);

  ResponseTopics.failed() : this(1, "", []);
}
```

- epic 定义,redux epic 可以看成是 action 的一个调度器,虽然 flutter 里的redux 也有 redux_thunk 中间件,但是 epic 这种基于流的调度中间件使得业务逻辑更加优雅

```dart
Stream<dynamic> fetchTopicsEpic(
    Stream<dynamic> actions, EpicStore<RootState> store) {
  return new Observable(actions)
      // 过滤特定请求
      .ofType(new TypeToken<RequestTopics>())
      .flatMap((action) {
        // 通过异步生成器来构建一个流
        return new Observable(() async* {
          try {
            // 发送获取主题列表的 http 请求
            final ret = await http.get("${apis['topics']}?page=${action.currentPage}&limit=6&tab=${action.category}&mdrender=false");
            Map<String, dynamic> result = json.decode(ret.body);
            List<Topic> topics = [];
            result['data'].forEach((v) {
              topics.add(new Topic.fromJson(v));
            });
            // 触发请求完成的回调,就是我们上面提到的 SmartRefresher widget 的复位
            action.afterFetched();
            yield new ResponseTopics(action.currentPage, action.category, topics);
          } catch(err) {
            print(err);
            yield new ResponseTopicsFailed(err);
          }
          // 刷新数据状态复位
          yield new ToggleLoading(false);
        } ());
  });
}
```

在接收到请求响应后,通过 `Topic.fromJson` 这个指定类构造器来创建主题列表,这个方法定义在 `store/model/topic.dart`里面

```dart
Topic.fromJson(final Map map):
    this.id = map["id"],
    this.authorName = map["author"]["loginname"],
    this.authorAvatar = map["author"]["avatar_url"],
    this.title = map["title"],
    this.tag = map["tab"],
    this.content = map["content"],
    this.createdAt = fromNow(map["create_at"]),
    this.lastReplyAt = fromNow(map["last_reply_at"]),
    this.replyCount = map["reply_count"],
    this.visitCount = map["visit_count"],
    this.top = map["top"],
    this.isCollect = map["is_collect"],
    this.replies = formatedReplies(map['replies']);
```

- `store/reducer/topic.dart`, 通过主题列表的 reducer 来变更 store 里面的数据状态

```dart
final Reducer<Map> topicsReducer = combineReducers([
  // 通过指定 action 类型来拆分
  new TypedReducer<Map, ClearTopic>(_clearTopic),
  new TypedReducer<Map, RequestTopics>(_requestTopics),
  new TypedReducer<Map, ResponseTopics>(_responseTopics)
]);

// 清空主题列表
Map _clearTopic(Map state, ClearTopic action) {
  return {};
}

Map _requestTopics(Map state, RequestTopics action) {
  Map topicsOfTopics = {};
  state.forEach((k, v) {
    final _v = new Map.from(v);
    if (action.category == k) {
      // 通过 isFetched 标志位来防止分类页面切换时重复请求
      _v["isFetched"] = false;
    }
    topicsOfTopics[k] = _v;
  });
  return topicsOfTopics;
}

Map _responseTopics(Map state, ResponseTopics action) {
  Map topicsOfCategory = {};
  state.forEach((k, v) {
    Map _v = {};
    _v.addAll(v);
    if (k == action.category) {
      List _list = [];
      // 上拉加载更多时
      if (_v['currentPage'] < action.currentPage) {
        _list.addAll(_v["list"]);
        _list.addAll(action.topics);
      } 
      // 下拉刷新时
      if (action.currentPage == 1) {
        _list.addAll(action.topics);
      }
      // 通过 isFetched 标志位来防止分类页面切换时重复请求
      _v["isFetched"] = true;
      _v["list"] = _list;
      _v["currentPage"] = action.currentPage;
    }
    topicsOfCategory[k] = _v;
  });
  return topicsOfCategory;
}
```

然后在 `store/reducer/root.dart` 的 rootReducer 里进行合并

```dart
RootState rootReducer(RootState state, action) {
  // 处理从持久化存储里加载数据状态
  if (action is PersistLoadedAction<RootState>) {
    return action.state ?? state;
  }
  // 将 state 里的数据状态对应到子 reducer
  return new RootState(
    tabIndex: tabReducer(state.tabIndex, action),
    auth:  loginReducer(state.auth, action),
    isLoading: loadingReducer(state.isLoading, action),
    topicsOfCategory: topicsReducer(state.topicsOfCategory, action),
    topic: topicReducer(state.topic, action),
    me: meReducer(state.me, action),
    collects: collectsReducer(state.collects, action),
    messages: messagesReducer(state.messages, action)
  );
}
```

- `store/index.dart` store 的初始化入口,在我们上面的入口widget里面使用 `StoreProvider` 容器包裹的时候传递

```dart
// 合并 epic 获得根 epic 提供给 epic 中间件调用
final epic = combineEpics([
  doLoginEpic, 
  fetchTopicsEpic, fetchTopicEpic, 
  fetchMeEpic,
  fetchCollectsEpic,
  fetchMessagesEpic,
  fetchMessageCountEpic,
  markAllAsReadEpic,
  markAsReadEpic,
  createReplyEpic,
  saveTopicEpic,
  createTopicEpic,
  toggleCollectEpic,
  likeReplyEpic,
]);

// 初始化持久化中间件存储容器
final persistor = Persistor<RootState>(
  storage: FlutterStorage('cnoder'),
  decoder: RootState.fromJson,
  debug: true
);

// 初始化 store
final store = new Store<RootState>(rootReducer,
  initialState: new RootState(), middleware: [
    new LoggingMiddleware.printer(), 
    new EpicMiddleware(epic),
    persistor.createMiddleware()
]);
```

这里有个小坑,持久化存储中间件 redux_persist 的文档上加载中间件的方式为

```dart
var store = new Store<AppState>(
  reducer,
  initialState: new AppState(),
  middleware: [persistor.createMiddleware()],
);
```

但是这样处理的话,在每个业务 action 触发的时候,都会触发持久化的操作,而这在很多场景下是不必要的,比如在我们的应用中只需要保存的用户身份令牌,所以只需要在触发登陆和登出 action 的时候执行持久化的操作,因此加载中间件的方式需要做如下改动

```dart
void persistMiddleware(Store store, dynamic action, NextDispatcher next) {
  next(action);
  // 仅处理登陆和登出操作
  if (action is FinishLogin || action is Logout) {
    try {
      persistor.save(store);
    } catch (_) {}
  }
}

// 初始化 store
final store = new Store<RootState>(rootReducer,
  initialState: new RootState(), middleware: [
    new LoggingMiddleware.printer(), 
    new EpicMiddleware(epic),
    persistMiddleware
]);
```

## 更多

应用的视图层和数据状态处理还是跟使用 React-Native 开发中使用 redux 技术栈的方式差不多,虽然整体目录结构有点繁琐,但是业务逻辑清晰明了,在后续功能扩展和维护的时候还是带来不少的方便,唯一遗憾的是因为 flutter 系统架构的问题,还没有一个针对 flutter 的 redux devtools,这一点还是蛮影响开发效率的

完整的项目源码请关注github仓库: [cnoder](https://github.com/ali322/cnoder),欢迎 star 和 PR,对 flutter 理解的不深,还望各位对本文中的不足之处批评指正
