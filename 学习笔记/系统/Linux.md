## Linux
**在Linux上添加定时任务的步骤**
> 1. 登录跳板机的ops用户或者测试服务器的root用户
> 2. `cd /home/ops/deploy/jar/shell`，将项目信息填写到appsinfo中，appsinfo中各列解释：Num(序号，部署是选择序号部署)，restart(部署后要不要部署脚本重启，1为重启，0为不重启)，project(项目名称)，arg(传入参数。一般为调用的类名，参数首字母小写)
> 3. 如果是新项目，而不是已有的项目添加一个新的参数，则需要在hostsinfo中添加主机信息，hostsinfo中各列解释，第一列项目名，第二列主机名
> 4. 运行addshell.sh,实例参数名称：项目名，参数：类名称（首字母小写）
> 5. 如果是测试环境需要退出root账户,(不要使用root账户进行部署项目，否则其他人使用ops账户部署会因部署权限问题无法拉取代码，无法重启任务)，使用ops账户部署task任务。
> 6. 如果需要启动测试服务器ops账户下执行/etc/init.d/$projectname_$argname start,如果是线上则先登录 Task01服务器，root账户启动任务
> 7. 定时执行需要`crontab -e`,和vim文本编辑一样，`* * * * *` 表示每分钟执行，   ```00 * * * *``` 表示每小时执行，`*/5 * * * *`

**Linux下grep显示多行信息：**

> * `grep -C 5 foo file`:显示file文件中匹配foo字符串那行以及上下5行
> * `grep -B 5 foo file`:显示foo以及前5行
> * `grep -A 5 foo file`:显示foo以及后5行

**从Linux服务器上拷贝数据到本地**：`scp root@ip:/home/a.txt /User/`将服务器的a.txt文件拷贝到本地/User目录下面
**grep同时满足多个关键字和满足任意关键字**

1. `grep -E "word1|word2|word3" file`满足任一条件（word1|word2|word3）将匹配
2. `grep word1 file|grep word2|grep word3`必须同时满足三个条件(word1,word2,word3)才匹配