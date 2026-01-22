# Jenkins 任务执行组件开发指南

本文档详细介绍如何开发一个支持动态参数的 Jenkins 任务执行组件。

## 目录

- [需求分析](#需求分析)
- [设计方案](#设计方案)
- [完整代码实现](#完整代码实现)
- [使用示例](#使用示例)
- [高级用法](#高级用法)

---

## 需求分析

### 功能需求

1. **固定参数**：
   - Jenkins 服务器地址
   - Job 名称
   - 认证信息（用户名、密码/Token）

2. **动态参数**：
   - 参数数量不固定（每个 Job 的参数不同）
   - 参数类型不固定（string、int、boolean 等）
   - 支持三种变量类型（plain、splice、lazy）

### 技术挑战

由于 Jenkins Job 的参数是动态的，我们需要：
- 使用灵活的数据结构来接收任意参数
- 支持参数的类型声明
- 保留变量的渲染类型信息

---

## 设计方案

### 方案一：使用 ObjectItemSchema（推荐）

使用对象类型来接收动态参数，每个参数包含 `value` 和 `type` 信息。

**优点**：
- 结构清晰，易于理解
- 支持参数类型声明
- 便于前端动态生成表单

**缺点**：
- 需要额外的类型字段

### 方案二：使用 ArrayItemSchema

使用数组来接收参数列表，每个元素包含 `key`、`value`、`type`。

**优点**：
- 更灵活，支持完全动态的参数
- 适合参数数量变化大的场景

**缺点**：
- 结构相对复杂
- 需要在代码中解析数组

---

## 完整代码实现

### 方案一：基于 ObjectItemSchema 的实现


```python
# ========== 使用组件时 ==========
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,  # ← 变量类型：告诉引擎需要渲染
    value=[
        {
            'key': 'branch',
            'value': '${git_branch}',  # ← 会被渲染
            'type': 'string'  # ← 数据类型：告诉 Jenkins 这是字符串参数
        },
        {
            'key': 'timeout',
            'value': '3600',
            'type': 'int'  # ← 数据类型：告诉 Jenkins 这是整数参数
        },
        {
            'key': 'enable_test',
            'value': 'true',
            'type': 'boolean'  # ← 数据类型：告诉 Jenkins 这是布尔参数
        }
    ]
)

# ========== 执行流程 ==========
# 步骤1：bamboo-engine 看到 Var.SPLICE，进行渲染
# ${git_branch} → 'master'

# 步骤2：渲染后的数据
[
    {'key': 'branch', 'value': 'master', 'type': 'string'},
    {'key': 'timeout', 'value': '3600', 'type': 'int'},
    {'key': 'enable_test', 'value': 'true', 'type': 'boolean'}
]

# 步骤3：execute() 方法中，根据 'type' 字段转换数据
def execute(self, data, parent_data):
    job_parameters = data.get_one_of_inputs('job_parameters')
    
    for param in job_parameters:
        key = param['key']
        value = param['value']
        param_type = param['type']  # ← 使用这个字段
        
        # 根据数据类型转换
        if param_type == 'int':
            value = int(value)  # '3600' → 3600
        elif param_type == 'boolean':
            value = value.lower() == 'true'  # 'true' → True
        
        # 传递给 Jenkins
        jenkins_params[key] = value

# 步骤4：调用 Jenkins API
# POST /job/deploy-app/buildWithParameters
# {
#     'branch': 'master',      # string 类型
#     'timeout': 3600,         # int 类型
#     'enable_test': True      # boolean 类型
# }
```

```python
# -*- coding: utf-8 -*-
"""
Jenkins 任务执行组件
支持动态参数，参数可以是 plain、splice、lazy 类型
"""

from pipeline.component_framework.component import Component
from pipeline.core.flow.activity import Service
from pipeline.core.flow.io import (
    InputItem,
    OutputItem,
    StringItemSchema,
    IntItemSchema,
    BooleanItemSchema,
    ObjectItemSchema,
)


class JenkinsJobExecuteService(Service):
    """
    Jenkins 任务执行服务
    
    该服务支持执行 Jenkins Job，并传递动态参数
    """
    
    def execute(self, data, parent_data):
        """
        执行 Jenkins Job
        
        注意：这里只是框架代码，实际执行逻辑需要根据业务需求实现
        """
        # 获取固定参数
        jenkins_url = data.get_one_of_inputs('jenkins_url')
        job_name = data.get_one_of_inputs('job_name')
        username = data.get_one_of_inputs('username')
        password = data.get_one_of_inputs('password')
        
        # 获取动态参数
        job_parameters = data.get_one_of_inputs('job_parameters')
        
        # TODO: 实现 Jenkins API 调用逻辑
        # 1. 连接 Jenkins 服务器
        # 2. 触发 Job 执行
        # 3. 传递参数
        # 4. 等待执行结果或返回 build number
        
        # 示例输出
        data.outputs.build_number = 123
        data.outputs.build_url = f"{jenkins_url}/job/{job_name}/123/"
        data.outputs.status = "SUCCESS"
        
        return True
    
    def inputs_format(self):
        """
        定义输入参数格式
        
        包含固定参数和动态参数
        """
        return [
            # ========== 固定参数 ==========
            InputItem(
                name='Jenkins 地址',
                key='jenkins_url',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='Jenkins 服务器地址，例如: http://jenkins.example.com'
                )
            ),
            InputItem(
                name='Job 名称',
                key='job_name',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='要执行的 Jenkins Job 名称，例如: deploy-production'
                )
            ),
            InputItem(
                name='用户名',
                key='username',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='Jenkins 认证用户名'
                )
            ),
            InputItem(
                name='密码/Token',
                key='password',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='Jenkins 认证密码或 API Token'
                )
            ),
            
            # ========== 动态参数 ==========
            InputItem(
                name='Job 执行参数',
                key='job_parameters',
                type='object',
                required=False,
                schema=ObjectItemSchema(
                    description='Jenkins Job 的执行参数，以键值对形式传递。'
                               '参数值会根据变量类型（plain/splice/lazy）自动渲染。'
                               '示例: {"branch": "master", "version": "1.0.0", "enable_test": true}',
                    property_schemas={}  # 空字典表示接受任意属性
                )
            ),
        ]

    def outputs_format(self):
        """
        定义输出参数格式
        """
        return [
            OutputItem(
                name='构建编号',
                key='build_number',
                type='int',
                schema=IntItemSchema(description='Jenkins Job 的构建编号')
            ),
            OutputItem(
                name='构建 URL',
                key='build_url',
                type='string',
                schema=StringItemSchema(description='Jenkins 构建的访问地址')
            ),
            OutputItem(
                name='执行状态',
                key='status',
                type='string',
                schema=StringItemSchema(
                    description='Job 执行状态',
                    enum=['SUCCESS', 'FAILURE', 'UNSTABLE', 'ABORTED']
                )
            ),
        ]


class JenkinsJobExecuteComponent(Component):
    """Jenkins 任务执行组件"""
    name = 'Jenkins 任务执行'
    code = 'jenkins_job_execute'
    bound_service = JenkinsJobExecuteService
```

---

### 方案二：基于 ArrayItemSchema 的实现（更灵活）

```python
# -*- coding: utf-8 -*-
"""
Jenkins 任务执行组件 - 数组参数版本
支持完全动态的参数配置
"""

from pipeline.component_framework.component import Component
from pipeline.core.flow.activity import Service
from pipeline.core.flow.io import (
    InputItem,
    OutputItem,
    StringItemSchema,
    IntItemSchema,
    BooleanItemSchema,
    ArrayItemSchema,
    ObjectItemSchema,
)


class JenkinsJobExecuteArrayService(Service):
    """
    Jenkins 任务执行服务 - 数组参数版本

    使用数组来接收动态参数，每个参数包含 key、value、type 信息
    """

    def execute(self, data, parent_data):
        """
        执行 Jenkins Job
        """
        # 获取固定参数
        jenkins_url = data.get_one_of_inputs('jenkins_url')
        job_name = data.get_one_of_inputs('job_name')
        username = data.get_one_of_inputs('username')
        password = data.get_one_of_inputs('password')

        # 获取动态参数数组
        job_parameters = data.get_one_of_inputs('job_parameters') or []

        # 将参数数组转换为字典
        params_dict = {}
        for param in job_parameters:
            key = param.get('key')
            value = param.get('value')
            param_type = param.get('type', 'string')

            # 根据类型转换值
            if param_type == 'int':
                try:
                    value = int(value)
                except (ValueError, TypeError):
                    pass
            elif param_type == 'boolean':
                value = str(value).lower() in ('true', '1', 'yes')

            params_dict[key] = value

        # TODO: 实现 Jenkins API 调用逻辑

        # 示例输出
        data.outputs.build_number = 123
        data.outputs.build_url = f"{jenkins_url}/job/{job_name}/123/"
        data.outputs.status = "SUCCESS"
        data.outputs.parameters_used = params_dict

        return True

    def inputs_format(self):
        """
        定义输入参数格式 - 使用数组接收动态参数
        """
        return [
            # ========== 固定参数 ==========
            InputItem(
                name='Jenkins 地址',
                key='jenkins_url',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='Jenkins 服务器地址，例如: http://jenkins.example.com'
                )
            ),
            InputItem(
                name='Job 名称',
                key='job_name',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='要执行的 Jenkins Job 名称'
                )
            ),
            InputItem(
                name='用户名',
                key='username',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='Jenkins 认证用户名'
                )
            ),
            InputItem(
                name='密码/Token',
                key='password',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='Jenkins 认证密码或 API Token'
                )
            ),

            # ========== 动态参数（数组形式） ==========
            InputItem(
                name='Job 执行参数列表',
                key='job_parameters',
                type='array',
                required=False,
                schema=ArrayItemSchema(
                    description='Jenkins Job 的执行参数列表，每个参数包含 key、value、type 信息',
                    item_schema=ObjectItemSchema(
                        description='单个参数配置',
                        property_schemas={
                            'key': StringItemSchema(description='参数名称'),
                            'value': StringItemSchema(description='参数值（会根据变量类型自动渲染）'),
                            'type': StringItemSchema(
                                description='参数数据类型',
                                enum=['string', 'int', 'boolean']
                            ),
                        }
                    )
                )
            ),
        ]

    def outputs_format(self):
        """
        定义输出参数格式
        """
        return [
            OutputItem(
                name='构建编号',
                key='build_number',
                type='int',
                schema=IntItemSchema(description='Jenkins Job 的构建编号')
            ),
            OutputItem(
                name='构建 URL',
                key='build_url',
                type='string',
                schema=StringItemSchema(description='Jenkins 构建的访问地址')
            ),
            OutputItem(
                name='执行状态',
                key='status',
                type='string',
                schema=StringItemSchema(
                    description='Job 执行状态',
                    enum=['SUCCESS', 'FAILURE', 'UNSTABLE', 'ABORTED']
                )
            ),
            OutputItem(
                name='使用的参数',
                key='parameters_used',
                type='object',
                schema=ObjectItemSchema(
                    description='实际传递给 Jenkins 的参数',
                    property_schemas={}
                )
            ),
        ]


class JenkinsJobExecuteArrayComponent(Component):
    """Jenkins 任务执行组件 - 数组参数版本"""
    name = 'Jenkins 任务执行（数组参数）'
    code = 'jenkins_job_execute_array'
    bound_service = JenkinsJobExecuteArrayService
```

---

## 使用示例

### 示例 1：使用 ObjectItemSchema 方案（推荐）

#### 场景：部署应用到生产环境

```python
from bamboo_engine.builder import Var, ServiceActivity, Data, EmptyStartEvent, EmptyEndEvent, builder

# 创建流程数据
pipeline_data = Data()

# 定义全局变量
pipeline_data.inputs['${git_branch}'] = Var(type=Var.PLAIN, value='master')
pipeline_data.inputs['${app_version}'] = Var(type=Var.PLAIN, value='v2.1.0')
pipeline_data.inputs['${deploy_env}'] = Var(type=Var.PLAIN, value='production')

# 创建 Jenkins 执行节点
jenkins_act = ServiceActivity(component_code='jenkins_job_execute')

# 设置固定参数
jenkins_act.component.inputs.jenkins_url = Var(
    type=Var.PLAIN,
    value='http://jenkins.example.com'
)
jenkins_act.component.inputs.job_name = Var(
    type=Var.PLAIN,
    value='deploy-application'
)
jenkins_act.component.inputs.username = Var(
    type=Var.PLAIN,
    value='admin'
)
jenkins_act.component.inputs.password = Var(
    type=Var.PLAIN,
    value='your-api-token-here'
)

# 设置动态参数 - 使用对象形式
# 方式 1：使用 Plain 类型（静态值）
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.PLAIN,
    value={
        'branch': 'master',
        'version': 'v2.1.0',
        'environment': 'production',
        'enable_rollback': True,
        'timeout': 3600,
    }
)

# 方式 2：使用 Splice 类型（引用全局变量）
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value={
        'branch': '${git_branch}',           # 引用全局变量
        'version': '${app_version}',         # 引用全局变量
        'environment': '${deploy_env}',      # 引用全局变量
        'enable_rollback': True,             # 静态值
        'timeout': 3600,                     # 静态值
    }
)
# 执行时会被渲染为：
# {
#     'branch': 'master',
#     'version': 'v2.1.0',
#     'environment': 'production',
#     'enable_rollback': True,
#     'timeout': 3600
# }

# 方式 3：混合使用（部分参数引用变量，部分使用静态值）
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value={
        'branch': '${git_branch}',
        'version': '${app_version}',
        'build_user': '${_system.operator}',  # 引用系统变量
        'build_time': '${_system.timestamp}', # 引用系统变量
        'notify_email': 'ops@example.com',    # 静态值
        'enable_test': False,                 # 静态值
    }
)
```

#### 场景：使用前一个节点的输出作为参数

```python
# 假设前一个节点输出了 git_commit_id
previous_act = ServiceActivity(component_code='git_get_commit')
# ... previous_act 执行后输出 commit_id

# Jenkins 节点引用前一个节点的输出
jenkins_act = ServiceActivity(component_code='jenkins_job_execute')
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value={
        'commit_id': '${commit_id}',         # 引用前一个节点的输出
        'branch': '${git_branch}',           # 引用全局变量
        'build_type': 'release',             # 静态值
    }
)
```

---

### 示例 2：使用 ArrayItemSchema 方案

#### 场景：动态配置参数列表

```python
from bamboo_engine.builder import Var, ServiceActivity

jenkins_act = ServiceActivity(component_code='jenkins_job_execute_array')

# 设置固定参数
jenkins_act.component.inputs.jenkins_url = Var(
    type=Var.PLAIN,
    value='http://jenkins.example.com'
)
jenkins_act.component.inputs.job_name = Var(
    type=Var.PLAIN,
    value='deploy-application'
)
jenkins_act.component.inputs.username = Var(type=Var.PLAIN, value='admin')
jenkins_act.component.inputs.password = Var(type=Var.PLAIN, value='token123')

# 设置动态参数 - 使用数组形式
# 方式 1：Plain 类型（静态参数列表）
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.PLAIN,
    value=[
        {'key': 'branch', 'value': 'master', 'type': 'string'},
        {'key': 'version', 'value': 'v2.1.0', 'type': 'string'},
        {'key': 'timeout', 'value': '3600', 'type': 'int'},
        {'key': 'enable_test', 'value': 'true', 'type': 'boolean'},
    ]
)

# 方式 2：Splice 类型（引用变量）
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value=[
        {'key': 'branch', 'value': '${git_branch}', 'type': 'string'},
        {'key': 'version', 'value': '${app_version}', 'type': 'string'},
        {'key': 'environment', 'value': '${deploy_env}', 'type': 'string'},
        {'key': 'timeout', 'value': '3600', 'type': 'int'},
    ]
)
# 执行时，value 字段中的变量会被自动渲染
```

---

## 高级用法

### 1. 使用 Lazy 变量动态生成参数

#### 场景：根据当前时间生成构建标签

```python
from pipeline.core.data.var import LazyVariable
import datetime

class BuildTagVariable(LazyVariable):
    """构建标签变量 - 自动生成带时间戳的标签"""
    code = 'build_tag'

    def get_value(self):
        # self.value 是 Splice 解析后的值（如版本号前缀）
        prefix = self.value or 'build'
        timestamp = datetime.datetime.now().strftime('%Y%m%d%H%M%S')
        return f"{prefix}-{timestamp}"

# 使用 Lazy 变量
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value={
        'branch': '${git_branch}',
        'build_tag': '${build_tag_var}',  # 引用 Lazy 变量
        'version': '${app_version}',
    }
)

# 在流程数据中定义 Lazy 变量
pipeline_data.inputs['${build_tag_var}'] = Var(
    type=Var.LAZY,
    custom_type='build_tag',
    value='release'  # 前缀
)
# 执行时会生成类似: 'release-20260122153045'
```

### 2. 条件参数传递

#### 场景：根据环境传递不同的参数

```python
# 定义环境变量
pipeline_data.inputs['${is_production}'] = Var(type=Var.PLAIN, value=True)

# 在组件中使用条件逻辑（需要在 execute 中处理）
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value={
        'branch': '${git_branch}',
        'environment': '${deploy_env}',
        # 注意：Mako 模板支持条件表达式
        'enable_rollback': '${"true" if is_production else "false"}',
        'timeout': '${7200 if is_production else 3600}',
    }
)
```

### 3. 参数验证和转换

在 Service 的 execute 方法中进行参数验证：

```python
def execute(self, data, parent_data):
    # 获取参数
    job_parameters = data.get_one_of_inputs('job_parameters') or {}

    # 参数验证
    required_params = ['branch', 'version', 'environment']
    missing_params = [p for p in required_params if p not in job_parameters]

    if missing_params:
        data.outputs.ex_data = f'缺少必需参数: {", ".join(missing_params)}'
        return False

    # 参数类型转换
    if 'timeout' in job_parameters:
        try:
            job_parameters['timeout'] = int(job_parameters['timeout'])
        except (ValueError, TypeError):
            data.outputs.ex_data = 'timeout 参数必须是整数'
            return False

    if 'enable_rollback' in job_parameters:
        job_parameters['enable_rollback'] = str(job_parameters['enable_rollback']).lower() in ('true', '1', 'yes')

    # TODO: 调用 Jenkins API

    return True
```

### 4. 完整的流程示例

```python
from bamboo_engine import api
from bamboo_engine.builder import (
    Var, Data, ServiceActivity,
    EmptyStartEvent, EmptyEndEvent, builder
)
from bamboo_engine.eri.runtime import BambooDjangoRuntime

# 1. 创建流程
start = EmptyStartEvent()

# 2. 获取代码节点
git_act = ServiceActivity(component_code='git_clone')
git_act.component.inputs.repository = Var(type=Var.PLAIN, value='https://github.com/example/app.git')
git_act.component.inputs.branch = Var(type=Var.PLAIN, value='master')

# 3. Jenkins 构建节点
jenkins_build = ServiceActivity(component_code='jenkins_job_execute')
jenkins_build.component.inputs.jenkins_url = Var(type=Var.PLAIN, value='http://jenkins.example.com')
jenkins_build.component.inputs.job_name = Var(type=Var.PLAIN, value='build-app')
jenkins_build.component.inputs.username = Var(type=Var.PLAIN, value='admin')
jenkins_build.component.inputs.password = Var(type=Var.PLAIN, value='token123')
jenkins_build.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value={
        'branch': 'master',
        'commit_id': '${commit_id}',  # 引用 git_act 的输出
        'build_type': 'release',
    }
)

# 4. Jenkins 部署节点
jenkins_deploy = ServiceActivity(component_code='jenkins_job_execute')
jenkins_deploy.component.inputs.jenkins_url = Var(type=Var.PLAIN, value='http://jenkins.example.com')
jenkins_deploy.component.inputs.job_name = Var(type=Var.PLAIN, value='deploy-app')
jenkins_deploy.component.inputs.username = Var(type=Var.PLAIN, value='admin')
jenkins_deploy.component.inputs.password = Var(type=Var.PLAIN, value='token123')
jenkins_deploy.component.inputs.job_parameters = Var(
    type=Var.SPLICE,
    value={
        'build_number': '${build_number}',  # 引用 jenkins_build 的输出
        'environment': 'production',
        'enable_rollback': True,
    }
)

# 5. 结束节点
end = EmptyEndEvent()

# 6. 连接节点
start.extend(git_act).extend(jenkins_build).extend(jenkins_deploy).extend(end)

# 7. 构建流程树
pipeline_data = Data()
pipeline = builder.build_tree(start, data=pipeline_data)

# 8. 执行流程
runtime = BambooDjangoRuntime()
api.run_pipeline(runtime=runtime, pipeline=pipeline)
```

---

## 最佳实践

### 1. 参数设计建议

✅ **推荐做法**：
- 使用 ObjectItemSchema 方案（方案一）更直观
- 固定参数和动态参数分开定义
- 为每个参数提供清晰的描述
- 使用 enum 限制参数的可选值

❌ **避免做法**：
- 不要在参数名中使用特殊字符
- 避免参数名过长
- 不要混淆参数的数据类型

### 2. 变量类型选择

| 场景 | 推荐类型 | 示例 |
|------|---------|------|
| 固定值 | Plain | `value='master'` |
| 引用全局变量 | Splice | `value='${git_branch}'` |
| 引用节点输出 | Splice | `value='${commit_id}'` |
| 动态计算 | Lazy | `custom_type='build_tag'` |
| 字符串拼接 | Splice | `value='v${major}.${minor}'` |

### 3. 错误处理

```python
def execute(self, data, parent_data):
    try:
        # 获取参数
        jenkins_url = data.get_one_of_inputs('jenkins_url')
        job_name = data.get_one_of_inputs('job_name')

        # 参数验证
        if not jenkins_url or not job_name:
            data.outputs.ex_data = 'Jenkins 地址和 Job 名称不能为空'
            return False

        # 调用 Jenkins API
        # ...

    except Exception as e:
        data.outputs.ex_data = f'执行失败: {str(e)}'
        return False

    return True
```

### 4. 安全建议

- ⚠️ **不要在代码中硬编码密码**，使用环境变量或密钥管理系统
- ⚠️ **使用 API Token** 而不是密码进行认证
- ⚠️ **对敏感参数进行加密**存储
- ⚠️ **记录操作日志**，便于审计

---

## 总结

本文档提供了两种 Jenkins 任务执行组件的实现方案：

1. **ObjectItemSchema 方案**（推荐）：
   - 使用对象接收动态参数
   - 结构清晰，易于使用
   - 适合大多数场景

2. **ArrayItemSchema 方案**：
   - 使用数组接收参数列表
   - 更灵活，支持完全动态配置
   - 适合参数变化大的场景

两种方案都支持：
- ✅ 动态参数数量
- ✅ 多种数据类型（string、int、boolean）
- ✅ 三种变量类型（plain、splice、lazy）
- ✅ 变量引用和模板渲染

选择合适的方案，根据实际业务需求进行定制开发！

### 5. 实现原理

Jenkins 任务执行组件的实现原理：

1. **组件定义**：使用 `Component` 类定义组件，包含 `Service` 服务和 `inputs_format` 输入格式定义。
2. **输入格式**：使用 `ObjectItemSchema` 或 `ArrayItemSchema` 定义动态参数格式。
3. **变量渲染**：使用 `Var.SPLICE` 类型的变量，bamboo-engine 会自动渲染 `${var}` 变量。
4. **数据类型转换**：在 `execute` 方法中，根据参数的 `type` 字段进行数据类型转换。
5. **调用 Jenkins API**：使用 `requests` 库调用 Jenkins API，传递渲染后的参数。

通过以上步骤，可以实现一个功能完善、可扩展的 Jenkins 任务执行组件。