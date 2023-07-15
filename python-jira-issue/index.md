# 利用python 查询操作JIRA中的issues


最近有个需求是获取某些符合条件的jira数据，在统计后使用echarts可视化度量出来。后端代码打算用 Python实现。
<!--more-->

## 问题解决
这里着重说下后台获取jira数据的代码。python有一个非常好用的jira操作库**jira-python**。[这里](https://jira.readthedocs.io/en/master/index.html)有其非常友好的文档说明。下面权当是对文档的摘抄复述吧！
### 安装
如果Python环境中集成了pip的话，可以直接使用如下命令安装：
```python
pip install jira
```
或者直接[下载](https://pypi.org/project/jira/#files)其离线包安装(注意，jira包在安装的时候可能有其他依赖包。其他依赖包也需要安装)这种安装比较麻烦，更推荐上面pip命令安装。
### 使用
我们知道jira里有相应的project、project下是对应的一些issues(我是这样理解的，一个jira是一个issue，issue也分类，project是同一类issues的统称。不知道这样说对不对，^_^)。要想获取jira对应的project或者issues，就得先建立一个jira的链接对象。即：
```python
from jira import JIRA
# 即通过jira的主网址、一个jira网站的用户名和密码即可获取该jira网站的链接对象
test_jira = JIRA('https://jirahost.com.cn', basic_auth=('username', 'password'))
```
下面就可以调用它来查询jira对应的projet与issues了。
#### 查询projet
```python
# 获取所有的projets
projects = test_jira.projects()
# 获取指定的project, JRA
test_proj = test_jira.project('JRA')
```
#### 查询issues
获取或查询指定条件的issues一般是用的最多也是最常用的。一般我们需要获取到issue，并对issue的数据进行统计度量。这里主要用到了这个包的**search_issues()**方法(该方法返回的是一个列表，里面都是符合对应条件的issue对象)，这个方法的参数是jira的**JQL**语句。**JQL**语句是一种用来查询jira数据的查询语句，类似于数据库的sql语句（例如mySQL的查询语句），但它不是数据库查询语句。[jira官网文档](https://confluence.atlassian.com/jiracoreserver073/advanced-searching-861257209.html)有更加详尽的说明。比如要查询名为“TEST”的project的issues，则其JQL语句为：
```
project = "TEST"
```
要查询指派人为“小明”，并且project名为“TEST”的issues，则其JQL语句为：
```
project = "TEST" AND assignee = "小明"
```
上面两句对应的Python代码分别为：
```python
search_str = 'project = "TEST"'
issues = test_jira.search_issues(search_str)
```
```python
search_str = 'project = "TEST" AND assignee = "小明"'
issues = test_jira.search_issues(search_str)
```
可以看到，要获取指定的jira或issues，只要写好对应的JQL语句即可。关于JQL的关键字的用法说明[jira官网文档](https://confluence.atlassian.com/jiracoreserver073/advanced-searching-861257209.html)都有。可以把符合条件的JQL语句写好，然后在自己或公司搭建的JIRA网站（即在首页导航栏的**Issues**--->**Search for issues**）中调试，如果可行，再放入代码中运行。
需要注意的是**search_issues()默认查询的issues最多为50个**。如果你查询的jira或issues数量超过50个，则需要添加**maxResults**参数来指定更大的值。例如获取project名为“TEST”的issues有100个，则Python中的代码为：

```python
# 指定获取的issues总数最多10000个
search_str = 'project = "TEST"'
issues = test_jira.search_issues(search_str, maxResults=10000)
```
获取到指定条件的issue列表，我们就可以用它来得到对应issue的详细信息了，比如最简单的我们想获取每个issue的名字，即：
```python
search_str = 'project = "TEST" AND assignee = "小明"'
# 得到issue的列表
issues = test_jira.search_issues(search_str)
# 获取issue的名字
name_list = [issue.key for issue in issues]
```
若是获取到对应的issue列表后，想获取issue的某一个信息，但我们又不知道其对应的是issue的哪个字段（有时候我们会给jira自定义某些字段），我们可以将获取到的issues转为ison数据，通过查看json数据，从而找到对应的字段。即：
```python
import json
search_str = 'project = "PROJ" AND Created="2019-3-18 8:12"'
# 用于json_result=True来返回对应的字典。
issue = test_jira.search_issues(search_str, json_result=True)
写入到json文件中
issue = json.dumps(issue, ensure_ascii=False)
with open('jira.json', 'w', encoding='utf-8') as f:
    f.write(issue)
    f.close()
```
之后将该json文件中的数据进行[代码格式化](http://json.parser.online.fr/)，找到自己想要的字段，进行后续的jira数据查询操作。

#### 删除、更新issues
方便起见，下面已将一些常用的功能进行了封装，代码如下：
```python
from jira import JIRA, JIRAError

server = JIRA(server='https://jirahost.com.cn',
              basic_auth=('username', 'password'))


# 查询issues
def search_issue(jql: str) -> list:
    """
    查询jira
    :param jql:
    :return:
    """
    issue_key_list = []
    try:
        issues = server.search_issues(jql, maxResults=10000)
        if len(issues) > 0:
            issue_key_list = [issue.key for issue in issues]
    except JIRAError as e:
        print(f'search issue failed: {e}')
    finally:
        return issue_key_list


# 验收issues.只有当issue的状态为待验收或验收中时才会验收
def close_jira(issue_key):
    """
    :param issue_key:
    :return:
    """
    status_list = ['待验收', '验收中']
    closed = False
    try:
        issue = server.issue(issue_key)
        issue_status = issue.fields.status.name
        if issue_status == '完成':
            return True

        if issue_status not in status_list:
            return False

        transitions = server.transitions(issue)
        # print(transitions)
        issue_id = transitions[0]['id']
        # print(issue_id)
        server.transition_issue(issue, issue_id)
        # 如果为待验收则递归一次
        if issue_status == '待验收':
            close_jira(issue_key)
        closed = True
    except JIRAError as e:
        print(f'close issue failed: {e}')
    finally:
        return closed


# 创建issue
def create_issue(issue_dict: dict):
    """
    创建一个issue,并返回其key
    :param issue_dict:
    :return: issue_key
    """
    issue_key = ''
    try:
        issue = server.create_issue(fields=issue_dict)
        issue_key = issue.key
    except JIRAError as e:
        print(f'create issue failed：{e}')
    finally:
        return issue_key


# 更新issue指派人
def modify_jira_assignee(issue_key: str, modify_usr: str):
    try:
        issue = server.issue(issue_key)
        issue.update(assignee=modify_usr)
    except JIRAError as e:
        print(e)


# 删除issue
def delete_jira(issue_key: str):
    try:
        issue = server.issue(issue_key)
        issue.delete()
    except JIRAError as e:
        print(e)

# 更改issue的截至日期
def modify_jira_duration(issue_key: str, duration: str):
    try:
        issue = server.issue(issue_key)
        issue.update(duedate=duration)
    except JIRAError as e:
        print(e)
```


---

> 作者: [JackBai](https://github.com/jackbai233)  
> URL: https://www.geekby.cn/python-jira-issue/  

