# Jenkins 组件核心代码示例

本文档提供 Jenkins 任务执行组件的核心 `inputs_format()` 实现代码。

---

## 方案一：ObjectItemSchema（推荐）- 对象形式接收动态参数

### 完整代码

```python
# -*- coding: utf-8 -*-
"""
Jenkins 任务执行组件 - 核心代码
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
    """Jenkins 任务执行服务"""
    
    def execute(self, data, parent_data):
        """执行逻辑（待实现）"""
        jenkins_url = data.get_one_of_inputs('jenkins_url')
        job_name = data.get_one_of_inputs('job_name')
        username = data.get_one_of_inputs('username')
        password = data.get_one_of_inputs('password')
        job_parameters = data.get_one_of_inputs('job_parameters')
        
        # TODO: 实现 Jenkins API 调用
        
        return True
    
    def inputs_format(self):
        """
        定义输入参数格式
        
        关键点：
        1. 固定参数：jenkins_url, job_name, username, password
        2. 动态参数：job_parameters (type='object', property_schemas={})
        3. property_schemas={} 表示接受任意键值对
        4. 参数值支持 plain/splice/lazy 三种变量类型
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
            
            # ========== 动态参数（核心） ==========
            InputItem(
                name='Job 执行参数',
                key='job_parameters',
                type='object',
                required=False,
                schema=ObjectItemSchema(
                    description='Jenkins Job 的执行参数，以键值对形式传递。'
                               '参数值会根据变量类型（plain/splice/lazy）自动渲染。'
                               '示例: {"branch": "master", "version": "1.0.0", "enable_test": true}',
                    property_schemas={}  # 空字典表示接受任意属性，这是关键！
                )
            ),
        ]
    
    def outputs_format(self):
        """定义输出参数格式"""
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

### 使用示例

```python
from bamboo_engine.builder import Var, ServiceActivity

# 创建组件实例
jenkins_act = ServiceActivity(component_code='jenkins_job_execute')

# 设置固定参数
jenkins_act.component.inputs.jenkins_url = Var(type=Var.PLAIN, value='http://jenkins.example.com')
jenkins_act.component.inputs.job_name = Var(type=Var.PLAIN, value='deploy-app')
jenkins_act.component.inputs.username = Var(type=Var.PLAIN, value='admin')
jenkins_act.component.inputs.password = Var(type=Var.PLAIN, value='token123')

# 设置动态参数 - Plain 类型（静态值）
jenkins_act.component.inputs.job_parameters = Var(
    type=Var.PLAIN,
    value={
        'branch': 'master',
        'version': 'v2.1.0',
        'enable_test': True,
        'timeout': 3600,
    }
)
```


