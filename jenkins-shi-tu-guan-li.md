# jenkins 视图管理
> 默认情况下jenkins 只有一个视图All, 所有的job 都在同一个视图下, 当自动化任务比较少时还好, 当自动化任务很多时, 一个视图看起来就很不舒服了, 此时就需要对任务进行分门别类, 不同类创建不同的视图了.

## 1. 默认All 视图
![](/assets/jenkins_2017-06-20_170247.png)

## 2. 创建自定义视图
* 点击上图中的加号, 输入视图名称, 选择List View , 点击OK
![](/assets/jenkins_2017-06-20_170415.png)

## 3. 设置视图
* 名称: 视图tab页签显示的名称
* 描述: 视图的描述, 显示在标题位置
* 正则表达式: 采用正则表达式筛选要显示的视图, 不推荐一个一个点击
* 点击添加列, 选择Description , 添加描述列. 这样在任务列表中就能显示出任务的描述信息了, 默认不支持显示描述列, 需要安装Description-cloumn 插件才行
* 拖动显示列, 可以调整显示列的顺序, 笔者习惯于将执行按钮放在最后一列.

![](/assets/jenkins_2017-06-20_171313.png)

## 3. LoadBalance 视图
![](/assets/jenkins_2017-06-20_170905.png)

## 4. 修改视图
* 修改: 点击左侧导航栏编辑按钮, 可以对视图设置进行修改
* 删除: 点击左侧导航栏中删除视图, 可以删除视图, 仅仅是删除视图而已, 不会删除任务