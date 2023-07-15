# python操作TFS的Work Item


因工作需要，现需要将jira切换到微软的TFS(Team Foundation Server)，并自动化创建TFS的任务(即 Work Item)。根据该需求，我首先使用了它的[REST API](https://docs.microsoft.com/zh-cn/rest/api/azure/devops/wit/work%20items?view=azure-devops-rest-6.0)进行尝试，但发现有些麻烦，后面找到了一个python库dohq-tfs，该库[文档](https://devopshq.github.io/tfs/installation.html)友好，操作简单方便，很适合快速的开发相应的脚本。
<!--more-->

## 条件准备
### 1. 安装库
使用pip安装dohq-tfs库，如下：

```
pip install dohq-tfs
```
### 2. 获取Token
登录到TFS系统，点击个人头像，找到安全性选项，在安全性选项下添加一个“个人访问令牌”，即个人token，复制该token信息，后续脚本中要用到。（经测试通过dohq-tfs库创建Work Item时直接使用用户和密码无法成功，而使用token却是可以的）

## 操作TFS

我们知道创建Work Item肯定是在TFS的某个项目下进行创建的。这里需要说明的是，无论是通过脚本创建work item还是获取work item，首先需要知道work item里面有哪些字段信息。我这里获取Work Item的字段信息方法是： **先手动在TFS上创建一个Work Item，然后获取该Work Item的API即可知道Work Item有哪些字段信息**。例如：我在TFS系统(网址为https://tfshost.com.cn/tfs/)的名为XB的项目下创建了一个任务项(Work Item)，该任务项的id为233，则该Work Item的api一般为：“https://tfshost.com.cn/tfs/XB/_apis/wit/workitems/233?api-version=1.0&api-version=1.0” （TFS[官网](https://docs.microsoft.com/zh-cn/rest/api/azure/devops/wit/work%20items/get%20work%20item?view=azure-devops-rest-6.0)也有Work Item的api组成介绍）。该api网址返回的是一些json格式的数据信息，具体如下：
```json
{
"id":233,
"rev":5,
"fields":{
"System.AreaPath":"XB",
"System.TeamProject":"XB",
"System.IterationPath":"XB",
"System.WorkItemType":"Task",
"System.State":"New",
"System.Reason":"New",
"System.AssignedTo":"张三",
"System.CreatedDate":"2020-03-09T02:16:28.847Z",
"System.CreatedBy":"张三",
"System.ChangedDate":"2020-03-09T07:01:45.317Z",
"System.ChangedBy":"张三",
"System.Title":"自动创建的任务",
"Microsoft.VSTS.Common.StateChangeDate":"2020-03-09T02:16:28.847Z",
"Microsoft.VSTS.Common.Priority":2,
"Microsoft.VSTS.Scheduling.FinishDate":"2020-03-25T16:00:00Z",
"Microsoft.VSTS.Scheduling.OriginalEstimate":20.0,
"Microsoft.VSTS.Scheduling.CompletedWork":20222.0,
"System.Description":"这是一个使用脚本自动创建tfs的任务",
"System.Tags":"测试"
},
...
}
```
上面只列出了一部分，通过上面信息我们知道，一个工作项中，状态信息对应的字段是：System.State，指派人对应的字段是：System.AssignedTo，标题对应的字段是：System.Title等等等等。知道了这些，我们就能得心应手的使用python库操作TFS了。
下面是对操作TFS的Work Item的一些总结，我只对一些常用功能进行了封装。（运行环境python3.6或以上）

```python
from tfs import TFSAPI, TFSClientError


# 声明在哪个项目下操作Work Item
project = 'XB'
# token信息
token = '64ugrjrfxwweazg5yd2cn3ulwm4tuqebdxkldyctabwihpqbnmpo'

client = TFSAPI('https://tfshost.com.cn/tfs/', project=project, pat=token)

# 获取某个work item 的信息
def get_tfs_fields(work_id: int):
    """
    :param work_id:
    :return:
    """
    workitem = client.get_workitem(work_id)
    # 获取workitem id
    # print(workitem.data['id'])
    field_names = workitem.field_names
    # print(workitem.revisions)
    for field_name in field_names:
        print(f'{field_name}:{workitem.get(field_name)}')


def create_tfs_task(fields: dict, work_type: str = 'Task') -> int:
    """
    创建一个work item
    :param work_type: 任务类型
    :param fields: {'System.Title': '自动创建任务',
          'System.Description': '这是一个使用脚本自动创建tfs的任务',
          'System.AssignedTo': '张三',
          }
    :return:
    """
    try:
        workitem = client.create_workitem(work_type, fields)
    except TFSClientError as e:
        print(f'error:{e}')
        return 0
    else:
        # print(workitem.get('AssignedTo'))
        # print(workitem.id)
        return int(workitem.id)


def add_child_task(parent_id: int, child_id: int):
    """
    给父任务添加子任务
    :param parent_id: 父任务ID
    :param child_id: 子任务ID
    :return:
    """
    try:
        parent_workitem = client.get_workitem(parent_id)
        child_workitem = client.get_workitem(child_id)

        parent_link_raw = [
            {
                "rel": "System.LinkTypes.Hierarchy-Reverse",
                "url": parent_workitem.url,
                "attributes": {
                    "isLocked": False
                }
            }
        ]
        child_workitem.add_relations_raw(relations_raw=parent_link_raw)
    except TFSClientError as e:
        print(f'error:{e}')
        return False
    else:
        return True


def update_tfs_task(work_id: int, data: dict):
    """
    更新work item 信息
    :param work_id:
    :param data:{"System.State": "进行中"}
    :return:
    """
    try:
        workitem = client.get_workitem(work_id)

        # workitem['System.State'] = "进行中"
        for key, value in data.items():
            workitem[key] = value
    except TFSClientError as e:
        print(f'error:{e}')


def search_tfs_task(title: str, work_type: str = 'Task'):
    """
    根据条件查询work item 信息，查询语句可根据自己需求定制，这里只是列了根据标题和任务类型查询
    :param work_type:
    :param title:
    :return:
    """
    query = f"""SELECT [System.Id],[System.WorkItemType]
        FROM workitems
        WHERE[System.WorkItemType] = '{work_type}' AND [System.Title] = '{title}'
        ORDER BY[System.ChangedDate]
        """
    try:
        wiql = client.run_wiql(query)

        ids = list(wiql.workitem_ids) or []
        # print(f"Found WI with ids={ids}")
        # # raw = wiql.result
        # workitems = wiql.workitems
        # print(workitems[0]['Title'])
    except Exception as e:
        print(f'error:{e}')
        return False
    else:
        return ids
```

更多操作方法请查看[官网](https://devopshq.github.io/tfs/installation.html)



---

> 作者: [JackBai](https://github.com/jackbai233)  
> URL: https://www.geekby.cn/python-tfs/  

