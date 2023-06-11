Newrootcmd-newApp->app.new



App.new 做的事情:

- Codec 创建

- interfaceRegistry 创建 🚩

- baseapp创建

  - 存储创建包括db,以及CommitMultiStore
  - 定义模块路由器，包含本机查询路由以及本机消息服务处理路由和grpc消息服务以及grpc消息处理路由

- 创建目前的kvstorekeys

- 初始化paramskeeper🚩

- add capability keeper and ScopeToModule for ibc module 🚩

- 添加各个模块的keeper

- 创建模块管理器

- app.mm.SetOrderBeginBlockers

- app.mm.SetOrderEndBlockers

- app.mm.SetOrderInitGenesis

- 通过模块管理器注册所有模块的不变量，如果某些值变了可能会触发链停

- 注册消息路由和查询路由，目前已经废弃掉了，使用RegisterServices

- 创建模块管理器配置器，配置msgServer和queryServer🚩

- 通过模块管理器注册所有模块的服务

  - 注册服务包括两个，一个是消息服务，一个是查询服务

    ```go
    // 第一个参数是服务定义（切片），第二个就是handlers，也就是对应的msgServer里面的定义个各个实现
    types.RegisterMsgServer(cfg.MsgServer(), keeper.NewMsgServerImpl(am.keeper))
    types.RegisterQueryServer(cfg.QueryServer(), am.keeper)
    ```

- 创建 SimulationManager

- 初始化存储

- 初始化Baseapp

  - app.SetInitChainer(app.InitChainer)
  - app.SetBeginBlocker(app.BeginBlocker)
  - app.SetAnteHandler(anteHandler)
  - 