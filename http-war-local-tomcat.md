# http 上获取war包, 本地tomcat

> http-local 模式是指, tomcat 和 jenkins 安装在同一台服务器上, war 存放在web 服务器上, 需要通过http 协议方式获取. 通过wget 命令将war包下载到本地之后, 就转换成了local-local模式了.

http-local 模式自动化部署逻辑:  
1. 通过wget 命令, 先将war包下载到jenkins 所在服务器上的指定目录, 如/tmp  
2. 将war包移动到工作空间中, 再拷贝到tomcat的 temp 目录下  
3. 执行重部署tomcat 脚本  
4. 重新部署成功之后, 执行备份脚本

## 1. 任务配置

点击jenkins ,新建自有风格的任务, 输入任务名称, 选中自有风格项目, 点击OK  
![](/assets/jenkins_2017-06-20_124131.png)

### 1.1 配置General

#### 1.1.1 配置-项目名称

笔者认为这个应该叫任务\(job\)名称更合适, 因为这个名称就是创建任务时填写的名称, jenkins用于标识它的内置变量也是 JOB\_NAME. 此名称一定要有一定的规则, 笔者命名为:LB-free-http-local

#### 1.1.2 配置-任务描述

对任务做一下言简意赅的描述, 可以配置显示在任务列表中.

#### 1.1.3 配置-旧的构建丢弃策略

指定旧的构建记录删除策略, 默认事不删除的, 这样会占用大量的磁盘空间, 没有太大意义. 笔者设置删除策略为: 保持构建的最大个数: 10, 也就是说最多保留10条构建记录

#### 1.1.4 配置-参数化构建过程

* 参数化构建过程是一个比较好用的功能, 脚本中可以使用定义的参数,但是jenkins 的插件却支持的不是特别好, 使用的时候需要注意一下.
* 对于local-local 模式的人物, 笔者会创建以下几个String 类型常用变量: ** 点击添加参数 -&gt; String Parameter **

| 参数名称 | 默认值 | 描述 |
| :--- | :--- | :--- |
| warName | LoadBalance | 项目war包名称,不包含后缀名 |
| warDir | /tmp | war 文件所目录, 绝对路径 |
| testUrl | [http://192.168.145.100:7080/LoadBalance/index.jsp](http://192.168.145.100:7080/LoadBalance/index.jsp) | tomcat 服务器测试地址 |
| serverHome | /opt/app/tomcat/tomcat-7-7080 | tomcat服务器路径, bin 目录的父目录 |
| timeout | 100 | 部署超时时间, 非整个job超时时间 |
| description |  | 重部署描述, 会写入备份文件 |

#### 1.1.5 配置-JDK

当系统设置中配置了多个jdk 时, 此时需要选择jdk 版本号, 笔者选择的是 jdk 1.7

### 1.2 构建

#### 1.2.1 从web 服务器上, 下载war包

构建模块中, 点击新增构建步骤 -&gt; Execute Shell

```
#!/bin/bash
#DESC 获取war 包脚本
#PARM 参数化参数: $warName, $warDir, $serverHome

#下载路径
url=http://192.168.145.100:81/wars/LoadBalance.war 

#输出日志
echo "[info] begin to download project: $warName"

#清楚原来的文件
rm -f $warDir/$warName.war

#下载war文件
wget $url -O $warDir/$warName.war
```

#### 1.2.2 上传war包到tomcat 服务器临时目录

构建模块中, 点击新增构建步骤 -&gt; Execute Shell

```bash
#!/bin/bash
#DESC 上传war包到tomcat 服务器的临时目录
#PARM 参数化参数: warDir, warName

#检测文件是否存在, 文件不存在, 直接退出构建
if [ ! -f "$warDir/$warName.war" ]; then
  echo "[error] The file $warDir/$warName.war is not exsits !!!"
  exit 1
else

  # 将war包移动到工作空间目录中
  echo "[info ] move $warDir/$warName.war to $WORKSPACE ..."
  rm -f $WORKSPACE/$warName.war 
  mv $warDir/$warName.war $WORKSPACE

  # 将工作空间中war包上传到tomcat服务器的temp 目录中
  echo "[info ] copy $warDir/$warName.war to $serverHome/temp ..."
  rm -f $serverHome/temp/$warName.war
  cp $WORKSPACE/$warName.war $serverHome/temp

fi
```

#### 1.2.3 重部署脚本

war包上传到tomcat 临时目录之后, 执行重新部署tomcat 脚本:  
0. 检测temp 目录中war 文件是否存在  
1. 停止 tomcat 服务器  
2. 删除webapps 目录中的war文件和文件夹  
3. 清空服务器工作目录  
4. 清空日志文件  
5. 将项目war包从临时目录\(temp\)移动到部署目录\(webapps\)  
6. 重新启动tomcat服务器  
7. 检测服务器是否能启动成功

构建模块中, 点击新增构建步骤 -&gt; Execute Shell

```bash
#!/bin/bash
#DESC 部署项目
#PARM 参数化参数: serverHome, testUrl, timeout, $warName

#输出日志
echo "[info ] begin deploy project: $warName"

##################### 常量定义 #####################
# 应用服务器相关文件夹
serverBin=$serverHome/bin
serverLog=$serverHome/logs
serverTemp=$serverHome/temp
serverDeploy=$serverHome/webapps
serverWork=$serverHome/work/Catalina/localhost


##################### 执行脚本 #####################
# 0. 检测文件是否存在
if [ ! -f "$serverTemp/$warName.war" ]; then
  echo "[error] The file $serverTemp/$warName.war is not exsits !!!"
  exit 1
else
   echo "[info ] Find file: $serverTemp/$warName.war";

fi

# 1. 关闭服务器
echo "[info ] shutdown the server ..."
ps -ef | grep -v grep | grep "$serverHome"| awk '{print $2}' | xargs kill -9

# 2. 删除部署项目的war包和文件夹
echo "[info ] delete the old project: $serverDeploy/$warName $serverDeploy/$warName.war"
rm -rf $serverDeploy/$warName $serverDeploy/$warName.war

# 3. 清空服务器工作目录
echo "[info ] clean the project work directory: $serverWork/$warName"
rm -rf $serverWork/$warName

# 4. 清空日志文件
echo "[info ] clean the server log: $serverLog/catalina.out"
echo "" > $serverLog/catalina.out

# 5. 将新项目文件移动到服务器部署文件夹中
echo "[info ] deploy the project: $serverTemp/$warName.war"
mv $serverTemp/$warName.war $serverDeploy

# 6. 重新启动服务器
echo "[info ] start the server: $serverBin/startup.sh &"
$serverBin/startup.sh &


# 7. 监控服务器是否启动成功
echo "[info ] monitor the server start ..."
cost=0
statusCode=0
while [ $statusCode -ne 200 -a $cost -le $timeout ]
  do
    statusCode=`curl -o /dev/null -s -w %{http_code} $testUrl`
    echo "cost:$cost ms, statusCode:$statusCode"
    cost=$(( $cost + 5 ))
    sleep 5
  done

if [ $statusCode -ne 200 ] ; then
  #如果启动不成功则杀死进程
  echo "[error] the server startup faild ! begin shutdown the server "
  ps -ef | grep -v grep | grep "$serverName" | awk '{print $2}' | xargs kill -9
  exit 1
else
  #服务器启动成功
  echo "[info ] the server startup successful !"

  #tomcat服务器在本地时,需要添加此限制
  BUILD_ID=dontKillMe bash $serverBin/startup.sh
  exit 0
fi
```

#### 1.2.4 备份项目脚本

重新部署成功之后, 对新版本进行备份

```bash
#!/bin/bash
#DESC 部署成功后,备份项目
#PARM 参数化参数: $warName, $warDir
#PARM jk内置参数: $JOB_NAME, $BUILD_NUMBER
#PARM 全局自定义: $ITEM_BACKUP, $ITEM_BID_FILE

#输出日志
echo "[info ] begin backup project: $warName"

# 备份文件夹
bk_dir=$ITEM_BACKUP/$JOB_NAME

# 创建备份文件夹: 如果文件夹不存在则创建文件夹, 否则删除原来的文件$warName.war
if [ ! -d "$bk_dir" ]; then
  mkdir -p $bk_dir
else
  rm -f $bk_dir/$warName.war
fi

# 备份文件
mv $WORKSPACE/$warName.war $bk_dir
cp $bk_dir/$warName.war $bk_dir/$warName.war.$BUILD_NUMBER

# 记录成功的id
date_time=`date "+%Y%m%d-%H%M"`
echo "$date_time $BUILD_NUMBER $description" >> $ITEM_BACKUP/$JOB_NAME/$ITEM_BID_FILE
```

## 2. 执行jenkins 任务

1. 点击 jenkins -&gt; LB-free-httpd-local -&gt;  Build with Parameters 
2. 输入部署描述信息, 点击立即构建
   ![](/assets/jenkins_2017-06-20_135845.png)
3. 点击版本号 \#1 右边的小三角, 会弹出菜单, 点击 console output, 可以查看日志输出

## 3. 测试:

### 3.1 测试

* 确定防火墙已关闭或者释放了tomcat 服务器端口7080
* 浏览器中输入测试地址:
  ![](/assets/jenkins_100_2017-06-20_135051.png)

### 3.2 查看备份

通过linux 远程工具登录Linux 服务器, 可以进入备份文件夹, 会发现新增了三个文件

* LoadBalance.war: 上次重部署成功的war包
* LoadBalance.war.1: 部署成功的war记录
* SUCCESSBID: 部署成功的记录

```bash
[admin@localhost backup]# pwd
/var/data/.jenkins/backup
[admin@localhost backup]# ls ./LB-free-http-local/
LoadBalance.war LoadBalance.war.1 SUCCESSBID
```

### 4. 注意:

* 新建httpd-local 模式任务时, 不仅需要修改参数化定义的值, 还需要修改下载war包中url的路径的值
* 可以将下载war包中的url 也做成参数化变量

## 附:完整配置示例

![](/assets/jenkins_http_local_2017-06-20_135605.png)

