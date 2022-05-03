最外层的Message是AspDisplayLog

1. 过滤掉 `AspDisplayLog.product != 39 || AspDisplayLog.baiduid is empty`
2. 过滤掉 `AspDisplayLog.timestamp is null || AspDisplayLog.log is null`
3. 根据 `AspDisplayLog.timestamp` 计算出 `day 和 sample_second`
4. 获取province_id

```c++
AspLogField asp_log_field = AspDisplayLog.asp_log.field；
BaseField base_field = asp_log_field.base_field;
// 请求级别的基础字段中拿到province_id
int province_id = base_field.field_added_by_asp.province_id
```

5. ```
   ```

6. 



// 数据源级别的字段

b2log::AspLogField* asp_log_field = asp_log->mutable_asp_log_field();

// 请求级别的基础字段

 b2log::BaseField* base_field = asp_log_field->mutable_base_field();

// 拿到省份id

 province_id = base_field->mutable_field_added_by_asp()->province_id();

// 遍历每个数据源

for each src_field in asp_log_field 

  src_field.charge_name not in (baidu_fc_gusuan、baidu_fc_tuiguangshikuang)

​     foreach ad_field in src_field->mutable_ad_field_list

​       userid = 























#### 2.1 针对单个任务

对于单个任务希望提供查询任务元信息、任务运行结果和触发任务的接口。

**接口1： 查询任务元信息**

入参：业务组，空间名称，任务名称

结果：

```plain
任务类型
依赖的数据源
// 以下针对Spark程序包任务举例
jar_name
jar_version
main class
参数1
参数2
...
```

**接口2： 查询指定时间范围内任务的全部运行结果**

入参：业务组，空间名称，任务名称，起始时间，截止时间

结果：

```plain
[
  {
    日期: 20220101,
    状态：成功 | 失败 | 运行中...
    其他信息: xxx
  }
]
```

**接口3： 操作任务**

入参: 

```plain
{
    业务组，
    空间名称，
    任务名称，
    行为类型：Trigger | Reset | Kill ...，
    dates: [
     {
         DATE: 20220101,
         参数1：xxx,
         参数2：xxx
     },
     {
        DATE: 20220101,
        参数1：xxx,
        参数2：xxx
     }
    ]
}
```

结果：触发是否成功等信息。

#### 2.2 针对DAG需求

如群里讨论，”空间回溯“可以作为虚拟DAG使用，我们继续了解它是否满足我们业务场景(我们有较多的写Baikaldb和palo的任务)，此刻假设凤阁满足DAG式任务调度，使用DAG名称来表示它。

**接口1： 查询DAG元信息**

入参：业务组，空间名称，DAG名称

结果：

```plain
[
    {
        task_meta: 单任务元信息,
        parent_tasks: [父任务],
        sub_tasks: [子任务]
    }
]
```

**接口2： 查询指定时间范围内DAG的全部运行结果**

入参：业务组，空间名称，DAG名称，起始时间，截止时间

结果：

```plain
[
  {
    日期: 20220101,
    状态：成功 | 失败 | 运行中...
    task_detail: [
      {
        // 任务1的运行结果
      },
      {
        // 任务2的运行结果
      }
    ]
  }
]
```

**接口3： 操作DAG**

入参: 

```plain
{
    业务组，
    空间名称，
    任务名称，
    行为类型：Trigger | Reset | Kill ...，
    dates: [
     {
         DATE: 20220101,
         tasks: [
            {
               // 任务1的参数
               参数1：xxx,
               参数2：xxx
            },
            {
               // 任务2的参数
               参数1：xxx,
               参数2：xxx
            }
         ]
     },
   ]
}
```



**接口5： 创建DAG(待定)**

入参：

```plain
{
    业务组，
    空间名称，
    DAG名称，
    DAG_task_detail: [
        {
            task_name: task在凤阁的唯一标识,
            parent_tasks: [父任务],
            sub_tasks: [子任务]
        }
    ]
}
```

结果：

```plain
{
    "status": 是否成功，
    "dag_meta": {
        // dag的meta信息
    }
}
```

凤阁需要提供的功能：

功能1：凤阁任务支持传递参数

功能2：自动重启失败的任务（本质上失败是高概率，因此自动重启有必要）

考虑是否开放查询类接口：查询空间回溯计划的结果；查询指定时间范围内任务的全部运行结果；

todo: **接口7: 启动空间回溯，支持按照时间段来回溯 (todo: 业务上能否接受凌晨定时启动 )**

