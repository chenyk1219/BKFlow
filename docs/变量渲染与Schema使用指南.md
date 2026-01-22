# 变量渲染与 Schema 使用指南

本文档详细介绍 bamboo-engine 中 InputItem、OutputItem 的渲染机制，以及如何在自定义组件中使用各种 Schema 类型。

## 目录

- [变量渲染机制](#变量渲染机制)
  - [变量类型概述](#变量类型概述)
  - [Plain 变量](#plain-变量)
  - [Splice 变量](#splice-变量)
  - [Lazy 变量](#lazy-变量)
  - [渲染流程](#渲染流程)
- [Schema 类型系统](#schema-类型系统)
  - [基础 Schema 类型](#基础-schema-类型)
  - [复合 Schema 类型](#复合-schema-类型)
- [自定义组件中使用 InputItem 和 OutputItem](#自定义组件中使用-inputitem-和-outputitem)
  - [基础用法](#基础用法)
  - [完整示例](#完整示例)

---

## 变量渲染机制

### 变量类型概述

bamboo-engine 支持三种变量类型，它们在执行时有不同的渲染行为：

| 变量类型 | 说明 | 是否需要渲染 | 使用场景 |
|---------|------|------------|---------|
| **plain** | 普通变量，直接返回值 | 否 | 静态值、常量 |
| **splice** | 模板变量，支持变量引用和拼接 | 是 | 引用其他变量、字符串拼接 |
| **lazy** | 延迟计算变量，支持自定义处理逻辑 | 是 | 需要动态计算的值 |

### Plain 变量

**Plain 变量**是最简单的变量类型，直接返回设置的值，不进行任何模板解析。

#### 实现原理

```python
class PlainVariable(Variable):
    def get(self):
        return self.value  # 直接返回原始值
```

#### 使用示例

```python
from bamboo_engine.builder import Var, ServiceActivity

act = ServiceActivity(component_code='example_component')
act.component.inputs.param = Var(type=Var.PLAIN, value='hello world')
```

**执行时**：`param` 的值就是 `'hello world'`，不会进行任何解析。

### Splice 变量

**Splice 变量**支持模板语法，可以引用流程上下文中的其他变量，使用 `${variable_name}` 格式。

#### 实现原理

```python
class SpliceVariable(Variable):
    def _build_reference(self, context):
        # 从模板中提取所有引用的变量，如 ${var1}
        keys = ConstantTemplate(self.value).get_reference()
        refs = {}
        for key in keys:
            refs[key] = OutputRef(format_constant_key(key), context)
        self._refs = refs
    
    def _resolve(self):
        # 解析所有引用的变量
        maps = {}
        for key in self._refs:
            ref_val = self._refs[key].value
            if issubclass(ref_val.__class__, Variable):
                ref_val = ref_val.get()
            maps[key] = ref_val
        # 使用 Mako 模板引擎渲染
        val = ConstantTemplate(self.value).resolve_data(maps)
        self._value = val
```

#### 使用示例

**示例 1：简单引用**

```python
# 在流程全局变量中定义
pipeline_data.inputs['${constant_1}'] = Var(type=Var.PLAIN, value='value_1')

# 在节点中引用
act.component.inputs.param_1 = Var(type=Var.SPLICE, value='${constant_1}')
```

**执行时**：`param_1` 的值会被解析为 `'value_1'`

**示例 2：字符串拼接**

```python
pipeline_data.inputs['${name}'] = Var(type=Var.PLAIN, value='张三')
pipeline_data.inputs['${age}'] = Var(type=Var.PLAIN, value=25)

act.component.inputs.message = Var(
    type=Var.SPLICE, 
    value='用户 ${name} 的年龄是 ${age} 岁'
)
```

**执行时**：`message` 的值会被解析为 `'用户 张三 的年龄是 25 岁'`

**示例 3：链式引用**

```python
pipeline_data.inputs['${constant_3}'] = Var(type=Var.PLAIN, value='value_3')
pipeline_data.inputs['${constant_2}'] = Var(type=Var.PLAIN, value='value_2_${constant_3}')
pipeline_data.inputs['${constant_1}'] = Var(type=Var.PLAIN, value='value_1_${constant_2}')

act.component.inputs.param = Var(type=Var.SPLICE, value='${constant_1}')
```

**执行时**：`param` 的值会被解析为 `'value_1_value_2_value_3'`

#### 模板语法支持

Splice 变量使用 **Mako 模板引擎**，支持：

- 变量引用：`${variable_name}`
- 表达式计算：`${1 + 2}`（结果为 3）
- 字符串操作：`${'hello'.upper()}`（结果为 'HELLO'）
- 嵌套访问：`${data['key']}`、`${list[0]}`

### Lazy 变量

**Lazy 变量**继承自 Splice 变量，在变量引用解析的基础上，允许开发者自定义额外的处理逻辑。

#### 实现原理

```python
class LazyVariable(SpliceVariable, metaclass=RegisterVariableMeta):
    def get(self):
        # 先执行 Splice 变量的解析
        self.value = super(LazyVariable, self).get()
        # 再执行自定义的处理逻辑
        return self.get_value()
    
    @abstractmethod
    def get_value(self):
        # 子类实现自定义逻辑
        pass
```

#### 自定义 Lazy 变量示例

```python
import datetime
from pipeline.core.data.var import LazyVariable

class CurrentTimeVariable(LazyVariable):
    code = 'current_time'  # 变量类型标识

    def get_value(self):
        # self.value 已经是 Splice 解析后的值
        # 可以作为时间格式字符串
        time_format = self.value or '%Y-%m-%d %H:%M:%S'
        return datetime.datetime.now().strftime(time_format)

# 注册后即可使用
```

**使用自定义 Lazy 变量**：

```python
act.component.inputs.timestamp = Var(
    type=Var.LAZY,
    custom_type='current_time',  # 对应 LazyVariable.code
    value='%Y年%m月%d日'
)
```

**执行时**：`timestamp` 的值会是当前时间，格式为 `'2026年01月22日'`

### 渲染流程

变量在节点执行时的渲染流程如下：

```
1. 获取节点数据 (Data)
   ├── inputs: Dict[str, DataInput]
   │   └── DataInput.need_render: bool  # 是否需要渲染
   │   └── DataInput.value: Any         # 变量值
   └── outputs: Dict[str, str]

2. 分离需要渲染和不需要渲染的输入
   ├── need_render_inputs = {k: v for k, v in inputs if v.need_render}
   └── render_escape_inputs = {k: v for k, v in inputs if not v.need_render}

3. 构建渲染上下文 (Context)
   ├── 收集流程全局变量
   ├── 收集父流程变量
   └── 收集前序节点输出

4. 使用 Template 渲染需要渲染的输入
   └── Template(need_render_inputs).render(context)

5. 合并渲染结果
   └── execute_inputs = rendered_inputs + render_escape_inputs

6. 传递给 Service.execute(data, parent_data)
```

#### 渲染规则

| 变量类型 | need_render | 渲染方式 |
|---------|------------|---------|
| plain | False | 不渲染，直接使用原始值 |
| splice | True | 使用 Mako 模板引擎渲染 |
| lazy | True | 先 Mako 渲染，再执行自定义逻辑 |

---

## Schema 类型系统

Schema 用于描述 InputItem 和 OutputItem 的数据类型、结构和约束，帮助前端生成表单、进行数据验证。

### 基础 Schema 类型

bamboo-engine 提供了以下基础 Schema 类型：

#### 1. IntItemSchema - 整数类型

```python
from pipeline.core.flow.io import IntItemSchema

schema = IntItemSchema(
    description='用户年龄',
    enum=[18, 20, 25, 30]  # 可选：枚举值限制
)

# 序列化后的结构
schema.as_dict()
# {
#     'type': 'int',
#     'description': '用户年龄',
#     'enum': [18, 20, 25, 30]
# }
```

#### 2. StringItemSchema - 字符串类型

```python
from pipeline.core.flow.io import StringItemSchema

schema = StringItemSchema(
    description='用户名称',
    enum=['admin', 'user', 'guest']  # 可选：枚举值限制
)

# 序列化后的结构
schema.as_dict()
# {
#     'type': 'string',
#     'description': '用户名称',
#     'enum': ['admin', 'user', 'guest']
# }
```

#### 3. FloatItemSchema - 浮点数类型

```python
from pipeline.core.flow.io import FloatItemSchema

schema = FloatItemSchema(
    description='商品价格',
    enum=[]  # 通常不设置枚举
)

# 序列化后的结构
schema.as_dict()
# {
#     'type': 'float',
#     'description': '商品价格',
#     'enum': []
# }
```

#### 4. BooleanItemSchema - 布尔类型

```python
from pipeline.core.flow.io import BooleanItemSchema

schema = BooleanItemSchema(
    description='是否启用'
)

# 序列化后的结构
schema.as_dict()
# {
#     'type': 'boolean',
#     'description': '是否启用',
#     'enum': []
# }
```

### 复合 Schema 类型

#### 1. ArrayItemSchema - 数组类型

ArrayItemSchema 用于描述数组，需要指定数组元素的 Schema。

```python
from pipeline.core.flow.io import ArrayItemSchema, StringItemSchema, IntItemSchema

# 字符串数组
string_array_schema = ArrayItemSchema(
    description='用户名列表',
    item_schema=StringItemSchema(description='用户名')
)

# 序列化后的结构
string_array_schema.as_dict()
# {
#     'type': 'array',
#     'description': '用户名列表',
#     'enum': [],
#     'items': {
#         'type': 'string',
#         'description': '用户名',
#         'enum': []
#     }
# }

# 整数数组
int_array_schema = ArrayItemSchema(
    description='端口列表',
    item_schema=IntItemSchema(description='端口号')
)
```

**嵌套数组示例**：

```python
# 二维数组：数组的数组
nested_array_schema = ArrayItemSchema(
    description='矩阵数据',
    item_schema=ArrayItemSchema(
        description='行数据',
        item_schema=IntItemSchema(description='单元格值')
    )
)

# 对应的数据结构：[[1, 2, 3], [4, 5, 6]]
```

#### 2. ObjectItemSchema - 对象类型

ObjectItemSchema 用于描述对象（字典），需要指定对象属性的 Schema。

```python
from pipeline.core.flow.io import (
    ObjectItemSchema,
    StringItemSchema,
    IntItemSchema,
    BooleanItemSchema
)

# 用户对象
user_schema = ObjectItemSchema(
    description='用户信息',
    property_schemas={
        'name': StringItemSchema(description='用户名'),
        'age': IntItemSchema(description='年龄'),
        'is_active': BooleanItemSchema(description='是否激活')
    }
)

# 序列化后的结构
user_schema.as_dict()
# {
#     'type': 'object',
#     'description': '用户信息',
#     'enum': [],
#     'properties': {
#         'name': {
#             'type': 'string',
#             'description': '用户名',
#             'enum': []
#         },
#         'age': {
#             'type': 'int',
#             'description': '年龄',
#             'enum': []
#         },
#         'is_active': {
#             'type': 'boolean',
#             'description': '是否激活',
#             'enum': []
#         }
#     }
# }
```

**复杂嵌套示例**：

```python
# 包含数组和嵌套对象的复杂 Schema
complex_schema = ObjectItemSchema(
    description='服务器配置',
    property_schemas={
        'hostname': StringItemSchema(description='主机名'),
        'port': IntItemSchema(description='端口'),
        'ssl_enabled': BooleanItemSchema(description='是否启用SSL'),
        'tags': ArrayItemSchema(
            description='标签列表',
            item_schema=StringItemSchema(description='标签')
        ),
        'metadata': ObjectItemSchema(
            description='元数据',
            property_schemas={
                'created_by': StringItemSchema(description='创建者'),
                'created_at': StringItemSchema(description='创建时间')
            }
        )
    }
)

# 对应的数据结构：
# {
#     'hostname': 'server01',
#     'port': 8080,
#     'ssl_enabled': True,
#     'tags': ['web', 'production'],
#     'metadata': {
#         'created_by': 'admin',
#         'created_at': '2026-01-22'
#     }
# }
```

---

## 自定义组件中使用 InputItem 和 OutputItem

### 基础用法

在自定义组件的 Service 中，通过 `inputs_format()` 和 `outputs_format()` 方法定义输入输出规范。

#### InputItem 参数说明

```python
InputItem(
    name='参数显示名称',      # 前端显示的名称
    key='param_key',         # 参数键名，用于在代码中访问
    type='string',           # 数据类型：'string', 'int', 'float', 'boolean', 'array', 'object'
    required=True,           # 是否必填，默认 True
    schema=StringItemSchema(description='参数说明')  # Schema 定义
)
```

#### OutputItem 参数说明

```python
OutputItem(
    name='输出显示名称',      # 前端显示的名称
    key='output_key',        # 输出键名，用于在代码中访问
    type='string',           # 数据类型
    schema=StringItemSchema(description='输出说明')  # Schema 定义
)
```

### 完整示例

#### 示例 1：简单的数学计算组件

```python
import math
from pipeline.component_framework.component import Component
from pipeline.core.flow.activity import Service
from pipeline.core.flow.io import (
    InputItem, OutputItem,
    IntItemSchema, FloatItemSchema, StringItemSchema
)

class MathCalculateService(Service):
    """数学计算服务"""

    def execute(self, data, parent_data):
        # 获取输入参数
        operation = data.get_one_of_inputs('operation')
        num1 = data.get_one_of_inputs('num1')
        num2 = data.get_one_of_inputs('num2')

        # 参数验证
        if not isinstance(num1, (int, float)) or not isinstance(num2, (int, float)):
            data.outputs.ex_data = '参数必须是数字类型'
            return False

        # 执行计算
        try:
            if operation == 'add':
                result = num1 + num2
            elif operation == 'subtract':
                result = num1 - num2
            elif operation == 'multiply':
                result = num1 * num2
            elif operation == 'divide':
                if num2 == 0:
                    data.outputs.ex_data = '除数不能为0'
                    return False
                result = num1 / num2
            elif operation == 'power':
                result = math.pow(num1, num2)
            else:
                data.outputs.ex_data = f'不支持的操作: {operation}'
                return False

            # 设置输出
            data.outputs.result = result
            data.outputs.message = f'{num1} {operation} {num2} = {result}'
            return True

        except Exception as e:
            data.outputs.ex_data = f'计算错误: {str(e)}'
            return False

    def inputs_format(self):
        return [
            InputItem(
                name='运算类型',
                key='operation',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='支持的运算类型',
                    enum=['add', 'subtract', 'multiply', 'divide', 'power']
                )
            ),
            InputItem(
                name='第一个数字',
                key='num1',
                type='float',
                required=True,
                schema=FloatItemSchema(description='第一个操作数')
            ),
            InputItem(
                name='第二个数字',
                key='num2',
                type='float',
                required=True,
                schema=FloatItemSchema(description='第二个操作数')
            )
        ]

    def outputs_format(self):
        return [
            OutputItem(
                name='计算结果',
                key='result',
                type='float',
                schema=FloatItemSchema(description='运算结果')
            ),
            OutputItem(
                name='结果描述',
                key='message',
                type='string',
                schema=StringItemSchema(description='计算过程描述')
            )
        ]

class MathCalculateComponent(Component):
    name = '数学计算'
    code = 'math_calculate'
    bound_service = MathCalculateService
```

**使用该组件**：

```python
from bamboo_engine.builder import Var, ServiceActivity

act = ServiceActivity(component_code='math_calculate')
act.component.inputs.operation = Var(type=Var.PLAIN, value='multiply')
act.component.inputs.num1 = Var(type=Var.PLAIN, value=10)
act.component.inputs.num2 = Var(type=Var.PLAIN, value=5)

# 执行后输出：
# {
#     'result': 50.0,
#     'message': '10 multiply 5 = 50.0',
#     '_result': True,
#     '_loop': 0
# }
```

#### 示例 2：用户管理组件（复杂 Schema）

```python
from pipeline.component_framework.component import Component
from pipeline.core.flow.activity import Service
from pipeline.core.flow.io import (
    InputItem, OutputItem,
    StringItemSchema, IntItemSchema, BooleanItemSchema,
    ArrayItemSchema, ObjectItemSchema
)

class UserManagementService(Service):
    """用户管理服务"""

    def execute(self, data, parent_data):
        # 获取输入
        action = data.get_one_of_inputs('action')
        user_info = data.get_one_of_inputs('user_info')

        # 模拟用户操作
        if action == 'create':
            # 创建用户逻辑
            user_id = 12345  # 模拟生成的用户ID
            data.outputs.user_id = user_id
            data.outputs.success = True
            data.outputs.message = f"成功创建用户: {user_info.get('username')}"

        elif action == 'update':
            # 更新用户逻辑
            data.outputs.success = True
            data.outputs.message = f"成功更新用户: {user_info.get('username')}"

        elif action == 'delete':
            # 删除用户逻辑
            data.outputs.success = True
            data.outputs.message = f"成功删除用户: {user_info.get('username')}"
        else:
            data.outputs.ex_data = f'不支持的操作: {action}'
            return False

        # 返回用户详情
        data.outputs.user_detail = {
            'username': user_info.get('username'),
            'email': user_info.get('email'),
            'age': user_info.get('age'),
            'is_active': user_info.get('is_active', True),
            'roles': user_info.get('roles', [])
        }

        return True

    def inputs_format(self):
        return [
            InputItem(
                name='操作类型',
                key='action',
                type='string',
                required=True,
                schema=StringItemSchema(
                    description='用户操作类型',
                    enum=['create', 'update', 'delete']
                )
            ),
            InputItem(
                name='用户信息',
                key='user_info',
                type='object',
                required=True,
                schema=ObjectItemSchema(
                    description='用户详细信息',
                    property_schemas={
                        'username': StringItemSchema(description='用户名'),
                        'email': StringItemSchema(description='邮箱地址'),
                        'age': IntItemSchema(description='年龄'),
                        'is_active': BooleanItemSchema(description='是否激活'),
                        'roles': ArrayItemSchema(
                            description='用户角色列表',
                            item_schema=StringItemSchema(description='角色名称')
                        )
                    }
                )
            )
        ]

    def outputs_format(self):
        return [
            OutputItem(
                name='用户ID',
                key='user_id',
                type='int',
                schema=IntItemSchema(description='创建的用户ID')
            ),
            OutputItem(
                name='操作成功',
                key='success',
                type='boolean',
                schema=BooleanItemSchema(description='操作是否成功')
            ),
            OutputItem(
                name='消息',
                key='message',
                type='string',
                schema=StringItemSchema(description='操作结果消息')
            ),
            OutputItem(
                name='用户详情',
                key='user_detail',
                type='object',
                schema=ObjectItemSchema(
                    description='用户完整信息',
                    property_schemas={
                        'username': StringItemSchema(description='用户名'),
                        'email': StringItemSchema(description='邮箱'),
                        'age': IntItemSchema(description='年龄'),
                        'is_active': BooleanItemSchema(description='激活状态'),
                        'roles': ArrayItemSchema(
                            description='角色列表',
                            item_schema=StringItemSchema(description='角色')
                        )
                    }
                )
            )
        ]

class UserManagementComponent(Component):
    name = '用户管理'
    code = 'user_management'
    bound_service = UserManagementService
```

**使用该组件**：

```python
from bamboo_engine.builder import Var, ServiceActivity

act = ServiceActivity(component_code='user_management')
act.component.inputs.action = Var(type=Var.PLAIN, value='create')
act.component.inputs.user_info = Var(
    type=Var.PLAIN,
    value={
        'username': 'zhangsan',
        'email': 'zhangsan@example.com',
        'age': 28,
        'is_active': True,
        'roles': ['admin', 'developer']
    }
)

# 执行后输出：
# {
#     'user_id': 12345,
#     'success': True,
#     'message': '成功创建用户: zhangsan',
#     'user_detail': {
#         'username': 'zhangsan',
#         'email': 'zhangsan@example.com',
#         'age': 28,
#         'is_active': True,
#         'roles': ['admin', 'developer']
#     },
#     '_result': True,
#     '_loop': 0
# }
```

#### 示例 3：结合变量引用的组件

```python
from bamboo_engine.builder import Var, ServiceActivity, Data, EmptyStartEvent, EmptyEndEvent, builder

# 定义全局变量
pipeline_data = Data()
pipeline_data.inputs['${db_host}'] = Var(type=Var.PLAIN, value='localhost')
pipeline_data.inputs['${db_port}'] = Var(type=Var.PLAIN, value=3306)
pipeline_data.inputs['${db_name}'] = Var(type=Var.PLAIN, value='mydb')

# 创建节点，使用 Splice 变量引用全局变量
act = ServiceActivity(component_code='database_connect')
act.component.inputs.connection_string = Var(
    type=Var.SPLICE,
    value='mysql://${db_host}:${db_port}/${db_name}'
)
# 执行时会被渲染为: 'mysql://localhost:3306/mydb'

# 使用前一个节点的输出
act2 = ServiceActivity(component_code='database_query')
act2.component.inputs.connection = Var(
    type=Var.SPLICE,
    value='${connection_id}'  # 引用前一个节点的输出
)
```

---

## 最佳实践

### 1. 选择合适的变量类型

- **静态值**：使用 `Var.PLAIN`
- **引用其他变量**：使用 `Var.SPLICE`
- **需要动态计算**：使用 `Var.LAZY`

### 2. Schema 设计原则

- **明确类型**：为每个 InputItem 和 OutputItem 指定准确的 type 和 schema
- **详细描述**：在 schema 的 description 中提供清晰的说明
- **使用枚举**：对于有限选项的参数，使用 enum 限制可选值
- **嵌套合理**：复杂数据结构使用 ObjectItemSchema 和 ArrayItemSchema 组合

### 3. 组件开发建议

- **参数验证**：在 execute 方法中验证输入参数的有效性
- **错误处理**：使用 `data.outputs.ex_data` 记录错误信息
- **输出完整**：确保 outputs_format 中声明的所有字段都有赋值
- **文档齐全**：为每个 InputItem 和 OutputItem 提供清晰的 description

### 4. 常见错误

#### 错误 1：Schema 类型与实际数据不匹配

```python
# ❌ 错误：声明为 int，但传入 string
InputItem(name='年龄', key='age', type='int', schema=IntItemSchema(description='年龄'))
act.component.inputs.age = Var(type=Var.PLAIN, value='25')  # 字符串而非整数

# ✅ 正确
act.component.inputs.age = Var(type=Var.PLAIN, value=25)  # 整数
```

#### 错误 2：忘记设置 required 参数

```python
# ❌ 错误：必填参数未标记
InputItem(name='用户名', key='username', type='string')  # required 默认为 True

# ✅ 正确：明确标记可选参数
InputItem(name='备注', key='remark', type='string', required=False)
```

#### 错误 3：ArrayItemSchema 缺少 item_schema

```python
# ❌ 错误：数组没有指定元素类型
ArrayItemSchema(description='标签列表')  # 缺少 item_schema

# ✅ 正确
ArrayItemSchema(
    description='标签列表',
    item_schema=StringItemSchema(description='标签')
)
```

---

## 总结

本文档详细介绍了 bamboo-engine 中的变量渲染机制和 Schema 类型系统：

1. **变量类型**：
   - Plain：直接返回值，不渲染
   - Splice：支持模板语法和变量引用
   - Lazy：在 Splice 基础上支持自定义处理逻辑

2. **Schema 类型**：
   - 基础类型：IntItemSchema、StringItemSchema、FloatItemSchema、BooleanItemSchema
   - 复合类型：ArrayItemSchema、ObjectItemSchema

3. **组件开发**：
   - 通过 inputs_format() 定义输入规范
   - 通过 outputs_format() 定义输出规范
   - 使用 Schema 提供类型约束和描述信息

掌握这些知识后，你就可以开发出功能强大、类型安全的自定义组件了！