# Pipeline Tree 结构详解

本文档详细说明 Jenkins 组件在 pipeline tree 中的数据结构。

---

## 📋 什么是 Pipeline Tree

Pipeline Tree 是 bamboo-engine 的核心数据结构，通过 `builder.build_tree()` 方法将流程定义转换为可执行的树形结构。

```python
from bamboo_engine.builder import builder

# 构建流程树
pipeline = builder.build_tree(start, data=pipeline_data)
```

---

## 🌳 Pipeline Tree 的整体结构

```json
{
    "id": "p1234567890",
    "start_event": { ... },
    "end_event": { ... },
    "activities": { ... },
    "gateways": { ... },
    "flows": { ... },
    "data": {
        "inputs": { ... },
        "outputs": [ ... ]
    }
}
```

---

## 🎯 Jenkins 组件在 Pipeline Tree 中的完整示例

### 场景：使用 ObjectItemSchema 方案

#### 1. 流程定义代码

```python
from bamboo_engine.builder import Var, ServiceActivity, Data, EmptyStartEvent, EmptyEndEvent, builder

# 创建流程数据
pipeline_data = Data()

# 定义全局变量
pipeline_data.inputs['${git_branch}'] = Var(type=Var.PLAIN, value='master')
pipeline_data.inputs['${app_version}'] = Var(type=Var.PLAIN, value='v2.1.0')

# 创建节点
start = EmptyStartEvent()
jenkins_act = ServiceActivity(component_code='jenkins_job_execute', name='部署应用')
end = EmptyEndEvent()

# 连接节点
start.extend(jenkins_act).extend(end)

# 设置 Jenkins 组件的输入参数
jenkins_act.component.inputs.jenkins_url = Var(
    type=Var.PLAIN,
    value='http://jenkins.example.com'
)
jenkins_act.component.inputs.job_name = Var(
    type=Var.PLAIN,
    value='deploy-app'
)
jenkins_act.component.inputs.username = Var(
    type=Var.PLAIN,
    value='admin'
)
jenkins_act.component.inputs.password = Var(
    type=Var.PLAIN,
    value='token123'
)

# 设置动态参数 - 使用 Splice 类型引用全局变量
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value={
        'branch': '${git_branch}',      # 引用全局变量
        'version': '${app_version}',    # 引用全局变量
        'timeout': 3600,                # 静态值
        'enable_test': True,            # 静态值
    }
)

# 构建 pipeline tree
pipeline = builder.build_tree(start, data=pipeline_data)
```

#### 2. 生成的 Pipeline Tree 结构

```json
{
    "id": "p1234567890abcdef",
    
    "start_event": {
        "id": "n001",
        "name": "",
        "type": "EmptyStartEvent",
        "incoming": "",
        "outgoing": "f001"
    },
    
    "end_event": {
        "id": "n003",
        "name": "",
        "type": "EmptyEndEvent",
        "incoming": "f002",
        "outgoing": ""
    },
    
    "activities": {
        "n002": {
            "id": "n002",
            "name": "部署应用",
            "type": "ServiceActivity",
            "incoming": "f001",
            "outgoing": "f002",
            "error_ignorable": false,
            "timeout": null,
            "skippable": true,
            "retryable": true,
            "optional": false,
            
            "component": {
                "code": "jenkins_job_execute",
                "version": "legacy",
                "inputs": {
                    "jenkins_url": {
                        "type": "plain",
                        "value": "http://jenkins.example.com"
                    },
                    "job_name": {
                        "type": "plain",
                        "value": "deploy-app"
                    },
                    "username": {
                        "type": "plain",
                        "value": "admin"
                    },
                    "password": {
                        "type": "plain",
                        "value": "token123"
                    },
                    "job_parameters": {
                        "type": "splice",
                        "value": {
                            "branch": "${git_branch}",
                            "version": "${app_version}",
                            "timeout": 3600,
                            "enable_test": true
                        }
                    }
                }
            }
        }
    },
    
    "flows": {
        "f001": {
            "id": "f001",
            "source": "n001",
            "target": "n002",
            "is_default": false
        },
        "f002": {
            "id": "f002",
            "source": "n002",
            "target": "n003",
            "is_default": false
        }
    },
    
    "gateways": {},
    
    "data": {
        "inputs": {
            "${git_branch}": {
                "type": "plain",
                "value": "master"
            },
            "${app_version}": {
                "type": "plain",
                "value": "v2.1.0"
            }
        },
        "outputs": []
    }
}
```

---

## 🔍 关键字段详解

### 1. `component.inputs` 字段

这是最关键的部分，存储了所有输入参数的**变量类型**和**值**：

```json
"inputs": {
    "job_parameters": {
        "type": "splice",  // ← 变量类型（plain/splice/lazy）
        "value": {         // ← 实际数据
            "branch": "${git_branch}",
            "version": "${app_version}",
            "timeout": 3600,
            "enable_test": true
        }
    }
}
```

**重要说明**：
- `"type": "splice"` 告诉引擎需要渲染这个变量
- `value` 中的 `${git_branch}` 会在执行时被替换
- `timeout` 和 `enable_test` 虽然在 splice 变量中，但因为不包含 `${}` 所以不会被渲染

### 2. `data.inputs` 字段

存储全局变量：

```json
"data": {
    "inputs": {
        "${git_branch}": {
            "type": "plain",
            "value": "master"
        },
        "${app_version}": {
            "type": "plain",
            "value": "v2.1.0"
        }
    }
}
```

---

## 🔄 执行时的渲染过程

### 步骤 1：引擎读取 Pipeline Tree

```python
# 引擎读取到 jenkins_act 节点
activity = pipeline['activities']['n002']
component_inputs = activity['component']['inputs']
```

### 步骤 2：检查变量类型

```python
job_parameters_var = component_inputs['job_parameters']
# {
#     "type": "splice",
#     "value": {
#         "branch": "${git_branch}",
#         "version": "${app_version}",
#         "timeout": 3600,
#         "enable_test": true
#     }
# }

# 引擎发现 type="splice"，需要进行渲染
```

### 步骤 3：渲染变量引用

```python
# 引擎查找全局变量
global_inputs = pipeline['data']['inputs']

# 渲染 ${git_branch}
# 1. 找到 global_inputs['${git_branch}'] = {"type": "plain", "value": "master"}
# 2. 替换 "${git_branch}" → "master"

# 渲染 ${app_version}
# 1. 找到 global_inputs['${app_version}'] = {"type": "plain", "value": "v2.1.0"}
# 2. 替换 "${app_version}" → "v2.1.0"

# 渲染后的结果
rendered_value = {
    "branch": "master",      # 已渲染
    "version": "v2.1.0",     # 已渲染
    "timeout": 3600,         # 未改变
    "enable_test": true      # 未改变
}
```

### 步骤 4：传递给 Service.execute()

```python
class JenkinsJobExecuteService(Service):
    def execute(self, data, parent_data):
        # data.get_one_of_inputs('job_parameters') 返回渲染后的值
        job_parameters = data.get_one_of_inputs('job_parameters')

        # job_parameters 的值：
        # {
        #     "branch": "master",
        #     "version": "v2.1.0",
        #     "timeout": 3600,
        #     "enable_test": true
        # }

        # 直接使用，无需再次解析
        print(job_parameters['branch'])  # 输出: master
```

---

## 📊 使用 ArrayItemSchema 方案的 Pipeline Tree

### 流程定义代码

```python
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value=[
        {'key': 'branch', 'value': '${git_branch}', 'type': 'string'},
        {'key': 'timeout', 'value': '3600', 'type': 'int'},
        {'key': 'enable_test', 'value': 'true', 'type': 'boolean'},
    ]
)
```

### Pipeline Tree 中的表示

```json
"component": {
    "code": "jenkins_job_execute_array",
    "inputs": {
        "job_parameters": {
            "type": "splice",
            "value": [
                {
                    "key": "branch",
                    "value": "${git_branch}",
                    "type": "string"
                },
                {
                    "key": "timeout",
                    "value": "3600",
                    "type": "int"
                },
                {
                    "key": "enable_test",
                    "value": "true",
                    "type": "boolean"
                }
            ]
        }
    }
}
```

### 渲染后的结果

```json
[
    {
        "key": "branch",
        "value": "master",  // ← ${git_branch} 被渲染
        "type": "string"
    },
    {
        "key": "timeout",
        "value": "3600",
        "type": "int"
    },
    {
        "key": "enable_test",
        "value": "true",
        "type": "boolean"
    }
]
```

---

## 🎨 可视化对比

### Plain 变量

```
定义时：
Var(type=Var.PLAIN, value={'branch': 'master'})

Pipeline Tree：
{
    "type": "plain",
    "value": {"branch": "master"}
}

执行时：
直接返回 {"branch": "master"}
```

### Splice 变量

```
定义时：
Var(type=Var.SPLICE, value={'branch': '${git_branch}'})

Pipeline Tree：
{
    "type": "splice",
    "value": {"branch": "${git_branch}"}
}

执行时：
1. 查找 ${git_branch} → "master"
2. 渲染后返回 {"branch": "master"}
```

### Lazy 变量

```
定义时：
Var(type=Var.LAZY, custom_type='build_tag', value='release')

Pipeline Tree：
{
    "type": "lazy",
    "custom_type": "build_tag",
    "value": "release"
}

执行时：
1. 找到 BuildTagVariable 类
2. 调用 get_value() 方法
3. 返回 "release-20260122153045"
```

---

## 🔑 关键要点总结

### 1. Pipeline Tree 中只存储变量类型，不存储数据类型

```json
// ✅ 正确：Pipeline Tree 中的结构
"job_parameters": {
    "type": "splice",  // ← 这是变量类型
    "value": {
        "timeout": 3600  // ← 这是数据（整数类型）
    }
}

// ❌ 错误：Pipeline Tree 中不会出现这样的结构
"job_parameters": {
    "type": "int",  // ← 数据类型不在这里
    "value": 3600
}
```

### 2. 数据类型在哪里？

数据类型体现在：
- **InputItem 的 Schema 定义中**（inputs_format）
- **实际数据的值中**（JSON 的原生类型）

```python
# Schema 定义（inputs_format）
InputItem(
    key='job_parameters',
    type='object',  // ← 这里定义数据类型
    schema=ObjectItemSchema(...)
)

# Pipeline Tree 中
"job_parameters": {
    "type": "splice",  // ← 变量类型
    "value": {
        "timeout": 3600  // ← 数据类型是 int（JSON 原生类型）
    }
}
```

### 3. ArrayItemSchema 方案中的 'type' 字段

```json
// 这是业务数据，不是变量类型
{
    "key": "timeout",
    "value": "3600",
    "type": "int"  // ← 这是告诉 Jenkins 的数据类型
}

// 整个数组的变量类型在外层
"job_parameters": {
    "type": "splice",  // ← 这才是变量类型
    "value": [ ... ]
}
```

---

## 📝 完整示例：从定义到执行

```python
# ========== 1. 定义流程 ==========
pipeline_data.inputs['${git_branch}'] = Var(type=Var.PLAIN, value='master')

jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value={'branch': '${git_branch}', 'timeout': 3600}
)

# ========== 2. 构建 Pipeline Tree ==========
pipeline = builder.build_tree(start, data=pipeline_data)

# Pipeline Tree 结构：
# {
#     "data": {
#         "inputs": {
#             "${git_branch}": {"type": "plain", "value": "master"}
#         }
#     },
#     "activities": {
#         "n002": {
#             "component": {
#                 "inputs": {
#                     "job_parameters": {
#                         "type": "splice",
#                         "value": {"branch": "${git_branch}", "timeout": 3600}
#                     }
#                 }
#             }
#         }
#     }
# }

# ========== 3. 执行时渲染 ==========
# 引擎自动渲染 splice 变量
# {"branch": "${git_branch}", "timeout": 3600}
#   ↓
# {"branch": "master", "timeout": 3600}

# ========== 4. 传递给 execute() ==========
def execute(self, data, parent_data):
    job_parameters = data.get_one_of_inputs('job_parameters')
    # job_parameters = {"branch": "master", "timeout": 3600}

    # 直接使用渲染后的值
    print(job_parameters['branch'])  # 输出: master
    print(job_parameters['timeout'])  # 输出: 3600
```

---

## 🎯 总结

1. **Pipeline Tree 中存储的是变量类型**（plain/splice/lazy），不是数据类型（string/int/bool）
2. **数据类型体现在 Schema 定义和实际值中**
3. **变量类型控制渲染行为**，数据类型控制数据格式
4. **执行时引擎会自动渲染 splice 和 lazy 变量**
5. **Service.execute() 接收到的是渲染后的值**

希望这个文档能帮助你理解 Pipeline Tree 的结构！🎉
