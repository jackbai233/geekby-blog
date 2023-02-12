# Django使用DataTables插件总结


Datatables插件是一款方便简单的展示数据的列表插件，本文介绍了怎样在Django中使用它。

<!--more-->

{{< admonition type=tip title="提示" open=true >}}
文章中例子已上传至[github](https://github.com/jackbai233/Django-Datatables-Example)
{{< /admonition >}}

## 基本使用
Datatables插件是一款方便简单的展示数据的列表插件。关于基本使用，[官方网站](https://datatables.net/examples/basic_init/zero_configuration.html)上的已介绍的很详细，这里我再稍微过一下。

 1. js配置。包含jquery和datatables的js
    ```javascript
        <script src="https://code.jquery.com/jquery-3.3.1.js"></script>
        <script stc="https://cdn.datatables.net/1.10.19/js/jquery.dataTables.min.js"></script>
    ```

 2. css配置。包含dataTables的css

    ```css
     <link href="https://cdn.datatables.net/1.10.19/css/jquery.dataTables.min.css" rel="stylesheet">
    ```

 3. html。初始化表格

    ```html
     <table id="example" class="display" style="width:100%">
            <thead>
                <tr>
                    <th>Name</th>
                    <th>Position</th>
                    <th>Office</th>
                    <th>Age</th>
                    <th>Start date</th>
                    <th>Salary</th>
                </tr>
            </thead>
            <tbody>
                <tr>
                    <td>Tiger Nixon</td>
                    <td>System Architect</td>
                    <td>Edinburgh</td>
                    <td>61</td>
                    <td>2011/04/25</td>
                    <td>$320,800</td>
                </tr>
                <tr>
                    <td>Garrett Winters</td>
                    <td>Accountant</td>
                    <td>Tokyo</td>
                    <td>63</td>
                    <td>2011/07/25</td>
                    <td>$170,750</td>
                </tr>
                <tr>
                    <td>Ashton Cox</td>
                    <td>Junior Technical Author</td>
                    <td>San Francisco</td>
                    <td>66</td>
                    <td>2009/01/12</td>
                    <td>$86,000</td>
                </tr>
            </tbody>
        </table>
    ```

 4. js配置。是表格dataTable化

    ```javascript
     <script type="text/javascript">
         $(document).ready(function() {
            $('#example').DataTable();
        } );
     </script>
    ```


## 与django结合使用
这里以一个展示用户姓名年龄的表格举例。假设数据库(数据库使用Django默认自带数据库)中有表格User,它的字段有name、age两项。
### 基本使用
> 基本使用的话，则是django作为后端，将要显示的数据传给DataTables进行展示。具体用法比较简单，DataTables的官网也很详细了。(官网文档都是第一份资料)
不多说，直接上代码
```python
def get_basic_tables(request):
    """
    创建基本的DataTables表格
    """
    user_list = []
    for user_info in User.objects.all():
        user_list.append({
            'name': user_info.name,
            'age': user_info.age
        })

    return render(request, 'example/basic_tables.html', {
        'users': user_list
    })
```
上面代码主要就是将数据取出并返回。
前端的展示代码如下：

```html
<table id="basic-table" class="table table-hover" width="100%">
    <thead>
        <tr>
            <th>学号</th>
            <th>姓名</th>
            <th>年龄</th>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>
```

```javascript
    $(document).ready(function () {
        $("#basic-table").DataTable({
            // 表下方页脚的类型，具体类别比较到，见[官网](https://datatables.net/examples/basic_init/alt_pagination.html)
            "pagingType": "simple_numbers",
            //启动搜索框
            searching: true,
            destroy : true,
            // 保存刷新时原表的状态
            stateSave: true,
            // 将显示的语言初始化为中文
            "language": {
                "lengthMenu": "选择每页 _MENU_ 展示 ",
                "zeroRecords": "未找到匹配结果--抱歉",
                "info": "当前显示第 _PAGE_ 页结果，共 _PAGES_ 页",
                "infoEmpty": "没有数据",
                "infoFiltered": "(获取 _MAX_ 项结果)",
                "paginate": {
                    "first": "首页",
                    "previous": "上一页",
                    "next": "下一页",
                    "last": "末页"
                }
            },
            // 此处重要，该data就是dataTables要展示的数据.users即为后台传递过来的数据
            data: {{ users | safe }},
            columns: [
                {
                    data: null,
                    width: "1%",
                    // 若想前端显示的不一样，则需要"render"函数
                    'render': function (data, type, full, meta) {
                        return meta.row + 1 + meta.settings._iDisplayStart;
                    }
                },
                {
                    data: "name",
                    'render': function (data, type, full, meta) {
                        return '<a class="text-warning" style="color:#007bff" title="年龄为'+ full.age +'">'+ data +'</a>';
                    }
                },
                {data: 'age'}
            ]
        })
    });
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019092823405112.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpYW9iYWlfb2w=,size_16,color_FFFFFF,t_70)
可以看到html中只初始化了表头，表的内容则在javascript中控制。最终显示出来的数据行，第一列是对表格数据的排序。从代码中看出，当data对应的数据被置为null时，单元格中的内容将由"render"对应的函数返回值决定。第一列datarender函数中meta.row相当于表格中行的索引，默认是从**0**开始，故为了学号显示从1开始，进行了加1操作。
render函数中的四个参数可谓是大有作用。 参数data刚好就是该函数上方“data”键对应的值的内容，比如第二列中的数据为‘data”键的值为name，则render函数中data就是name。而参数full相当于后端传递过来的users中的每个user的索引，这样某一个单元格的内容想与它所在行的其他单元格进行互动，则可用full参数来传递。表格中当使用鼠标移动到名字上时，会显示到该人名的年龄，这一功能就是使用了full：

```javascript
   {
       data: "name",
       'render': function (data, type, full, meta) {
           return '<a class="text-warning" style="color:#007bff" title="年龄为'+ full.age +'">'+ data +'</a>';
       }
   },
```
### ajax请求数据使用

#### 基本操作
这种使用方法，则是前端发送ajax请求去后端获取数据，而不是一开始就有后端将数据传送到前端的。当数据再由后端传递回前端时，前端会自己进行处理，如分页等。下面例子是展示年龄为22周岁的人员表格
```html
<table id="ajax-table" class="table table-hover" width="100%">
    <thead>
        <tr>
            <th>学号</th>
            <th>姓名</th>
            <th>年龄</th>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>
```

html页面中依旧只是初始化了表头。

```javascript
    $(document).ready(function () {
        //django post请求需要加认证，不能忘了
        $.ajaxSetup({
            data: {csrfmiddlewaretoken: '{{ csrf_token }}' }
        });

        var table = $('#ajax-table').DataTable({
            "pagingType": "full_numbers",
            // 跟基本使用对比，已经没有data属性，而是多了"ajax"
            "ajax":{
                "processing": true,
                // ajax请求的网址
                "url": "{% url 'example:request_ajax' %}",
                "type": 'POST',
                "data": {
                    // 前端向后端传递的数据age,比如只查询年龄在22岁的人员
                    "age": 22
                },
                //
                "dataSrc": ""
            },
            // ajax请求成功传递回来后数据的展示
            columns: [
               {
                    data: null,
                    width: "1%",
                    // 若想前端显示的不一样，则需要"render"函数
                    'render': function (data, type, full, meta) {
                        return meta.row + 1 + meta.settings._iDisplayStart;
                    }
                },
                {
                    data: "name",
                    'render': function (data, type, full, meta) {
                        return '<a class="text-warning" style="color:#007bff" title="年龄为'+ full.age +'">'+ data +'</a>';
                    }
                },
                {data: 'age'}
            ],
            "language": {
                "processing": "正在获取数据，请稍后...",
                "lengthMenu": "选择每页 _MENU_ 展示 ",
                "zeroRecords": "未找到匹配结果--抱歉",
                "info": "当前显示第 _PAGE_ 页结果，共 _PAGES_ 页, 共 _TOTAL_ 条记录",
                "infoEmpty": "没有数据",
                "infoFiltered": "(获取 _MAX_ 项结果)",
                "sLoadingRecords": "载入中...",
                "paginate": {
                    "first": "首页",
                    "previous": "上一页",
                    "next": "下一页",
                    "last": "末页"
                }
            }
        } );
    });
```
后端代码处理ajax请求:
```python
def request_ajax(request):
    """
    处理ajax的例子中的post请求
    :param request:
    :return:
    """
    try:
        if request.method == "POST":
            # print(request.POST)
            #  获取到前端页面ajax传递过来的age
            age = int(request.POST.get('age', 22))

            user_list = []
            for user_info in User.objects.filter(age=age):
                user_list.append({
                    'name': user_info.name,
                    'age': user_info.age
                })

            # 主要是将数据库查询到的数据转化为json形式返回给前端
            return HttpResponse(json.dumps(user_list), content_type="application/json")
        else:
            return HttpResponse(f'非法请求方式')
    except Exception as e:
        return HttpResponse(e.args)
```
相比来看跟基本使用没多少区别，只是多了一步ajax请求而已。

#### 后端分页
当我们要往前端展示的数据量过大时，如果还是一股脑将数据全部扔给前端来处理，那么你会发现前端分页加载的性能很差，这时我们可以将分页操作放到后端来做。
其实将分页放到后端的意思就是**对后台数据库中的数据进行部分请求**。我们首先可以这样想：**“用户在前端页面查看表格时，他其实只关心这一页数据，他看不到其他页的数据。要看到其他页的数据，他必须得点击网页中的上一页或下一页按钮。”** 理解了这一点，我们是否可以这样做，即：**“用户想看哪一页的数据，我就只去后台数据库查询这一页的数据。”** 有了这样的理解，下来就是具体操作了。这个思路其实类似与Python语法中列表的**切片**功能，例如：
```python
test_list = [1, 3, 4, 5, 6, 7, 8, 9, 10, 11]
# 测试我们只需要这个列表中第3个到6个这4条数据，那么用列表的切片
little_list = test_list[2:6]
```
要查询到某一页展示的数据是哪些，必须知道这一页的数据的对应的数据起始位置和结束位置。
恰好在DataTables中，每次用户点击翻页（上一页或下一页）按钮时，**前端都会向后端发送一次ajax请求。**  而这次请求，前端则会将这一页的起始位置与结束位置传递到后端。
下面上代码：

```html
<table id="basic-table" class="table table-hover" width="100%">
    <thead>
        <tr>
            <th>学号</th>
            <th>姓名</th>
            <th>年龄</th>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>
```

```javascript
    $(document).ready(function () {
        //django post请求需要加认证，不能忘了
        $.ajaxSetup({
            data: {csrfmiddlewaretoken: '{{ csrf_token }}' }
        });

        var table = $('#backend-table').DataTable({
            "pagingType": "full_numbers",
            // 跟基本使用对比，已经没有data属性，而是多了"ajax"
            searching: false,
            destroy: true,
            stateSave: true,
            // 此处为ajax向后端请求的网址
            sAjaxSource: "{% url 'example:request_backend' %}",
            "processing": false,
            "serverSide": true,
            "bPaginate" : true,
            "bInfo" : true, //是否显示页脚信息，DataTables插件左下角显示记录数
            "sDom": "t<'row-fluid'<'span6'i><'span6'p>>",//定义表格的显示方式
            //服务器端，数据回调处理
            "fnServerData" : function(sSource, aoData, fnCallback) {
                $.ajax({
                    "dataType" : 'json',
                    // 此处用post，推荐用post形式，get也可以，但可能会遇到坑
                    "type" : "post",
                    "url" : sSource,
                    "data" : aoData,
                    "success" : function(resp){
                        fnCallback(resp);
                    }
                });
            },
            // ajax请求成功传递回来后数据的展示
            columns: [
               {
                    data: null,
                    width: "1%",
                    // 若想前端显示的不一样，则需要"render"函数
                    'render': function (data, type, full, meta) {
                        return meta.row + 1 + meta.settings._iDisplayStart;
                    }
                },
                {
                    data: "name",
                    'render': function (data, type, full, meta) {
                        return '<a class="text-warning" style="color:#007bff" title="年龄为'+ full.age +'">'+ data +'</a>';
                    }
                },
                {data: 'age'}
            ],
            "language": {
                "processing": "正在获取数据，请稍后...",
                "lengthMenu": "选择每页 _MENU_ 展示 ",
                "zeroRecords": "未找到匹配结果--抱歉",
                "info": "当前显示第 _PAGE_ 页结果，共 _PAGES_ 页, 共 _TOTAL_ 条记录",
                "infoEmpty": "没有数据",
                "infoFiltered": "(获取 _MAX_ 项结果)",
                "sLoadingRecords": "载入中...",
                "paginate": {
                    "first": "首页",
                    "previous": "上一页",
                    "next": "下一页",
                    "last": "末页"
                }
            }
        } );
        // 每隔5秒刷新一次数据
        // setInterval(refresh, 5000);
    });
    function refresh() {
        var table = $('#backend-table').DataTable();
        table.ajax.reload(null, false); // 刷新表格数据，分页信息不会重置
    }
```
后端代码：
```python
# 暂时跳过csrf的保护
@csrf_exempt
def request_backend(request):
    """
    处理后端分页例子中的post请求
    :param request:
    :return:
    """
    try:
        if request.method == "POST":
            # 获取翻页后该页要展示多少条数据。默认为10；此时要是不清楚dataTables的ajax的post返回值
            # 可以打印一下看看print(request.POST)
            page_length = int(request.POST.get('iDisplayLength', '10'))
            # 该字典将转化为json格式的数据返回给前端，字典中的key默认的名字不能变
            rest = {
                "iTotalRecords": page_length,  # 本次加载记录数量
            }
            # 获取传递过来的该页的起始位置，第一页的起始位置为0.
            page_start = int(request.POST.get('iDisplayStart', '0'))
            # 该页的结束位置则就是"开始的值 + 该页的长度"
            page_end = page_start + page_length
            # 开始查询数据库，只请求这两个位置之间的数据

            users = User.objects.all()[page_start:page_end]
            total_length = User.objects.all().count()

            user_list = []
            for user_info in users:
                user_list.append({
                    'name': user_info.name,
                    'age': user_info.age
                })
            # print(start, ":", length, ":", draw)
            # 此时的key名字就是aaData，不能变
            rest['aaData'] = user_list
            # 总记录数量
            rest['iTotalDisplayRecords'] = total_length
            return HttpResponse(json.dumps(rest), content_type="application/json")
        else:
            return HttpResponse(f'非法请求方式')
    except Exception as e:
        return HttpResponse(e.args)
```
如果我们想让前端实现定时刷新，可以用js的**setInterval**方法。具体如下：

```javascript
function refresh() {
        var table = $('#backend-table').DataTable();
        table.ajax.reload(null, false); // 刷新表格数据，分页信息不会重置
}
// 每隔5秒刷新一次数据
setInterval(refresh, 5000);
```

## 最后
以上的还有好多的dataTables插件的方法没有使用到，[官网](https://datatables.net/examples/basic_init/zero_configuration.html)有很多的使用事例，也非常详细，此处也只是把常用到的进行了归纳。这里多说一句，关于ajax请求后DataTables默认的查询会不起作用，但它会将查询框中的text通过ajax返回，需要自己在后端进行处理。
以上内容若有错误，请及时指正哈！
