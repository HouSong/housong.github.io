#azkaban 使用事项

## 开源实现的文档
我们的azkaban的使用方法和开源基本相似，关于flow的创建、管理和启动，以及页面的使用，参见[官方文档](http://azkaban.github.io/azkaban/docs/latest/)。

## 与开源实现的不同
为了满足我们内部的需要，我进行了一些定制化，主要是多机执行、权限控制、参数生成和传递等。

### host使用
在首页的Executors页面上查看自己机器对应的`executor id`并记录。启动flow的时候需要在`flow parameter`里面加上`useExecutor`，并将记录的id写到后面的值中。不写的话将无法进行调度。之后会在job中增加这个`useExecutor`，可以用`flow parameter`进行覆盖，目前只能手动指定。

### 启动用户
在executor上，将使用启动这个flow的用户来作为启动每个job进程的用户（使用类似`su`的命令），如果这个executor的机器上没有这个用户，那这个job将会出错。

### 参数传递（外部传入以及内部生成后的传递）
可以在启动flow的时候，指定`flow parameter`，会覆盖掉之前的flow定义中的变量值，job能在环境变量里看到这些参数。每个job都有一个环境变量叫`JOB_OUTPUT_PROP_FILE`，它指向一个本地文件，可以用json格式把新生成的参数写到里面，这些参数会传递到后面的所有节点里，能在各自的环境变量里看到。比如，如果整个flow流程需要`date`变量，可以在第一个节点中查看`date`变量，如果没有就按照当前时间生成；如果已经存在，说明是`flow parameter`中带入的，需要跑指定的日期的数据。这个功能在重跑历史数据的时候，可以复用线上的计算逻辑。

### 可用的jobtype
只能用built-in的那几种，目前只测试过`command`类型的任务。
