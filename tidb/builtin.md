# TiDB 增加 MySQL 内建函数

本文档用于描述如何为 TiDB 新增 builtin 函数。首先介绍一些必需的背景知识，然后介绍增加 builtin 函数的流程，最后会以一个函数作为示例。

### **背景知识**

SQL 语句在 TiDB 中是如何执行的。

SQL 语句首先会经过 parser，从文本 parse 成为 AST（抽象语法树），再经过 planner 指定执行计划以及 optimizer 对执行计划的优化，得到一个可以执行的 plan (执行计划)，通过执行这个 plan 即可得到结果，这期间会涉及到如何获取 table 中的数据，如何对数据进行过滤、计算、排序、聚合、滤重等操作。对于一个 builtin 函数，比较重要的是 parse 和如何求值。这里着重说这两部分。

#### Parse

TiDB 使用 yacc/lex 作为语法解析工具，如果对此完全不了解，可以先看一些教程，如：[https://www.ibm.com/developerworks/cn/linux/sdk/lex/](https://www.ibm.com/developerworks/cn/linux/sdk/lex/) 。语法解析的代码在 parser 目录下的，scanner.l 和 parser.y 两个文件，通过 goyacc/golex，可以将其转换为 parser.go 和 scanner.go 两个 go 代码文件。转换后的 go 代码，可以被其他的 go 代码调用，执行 parse 操作。

 

将 sql 语句从文本 parse 成结构化的过程中，首先是通过 scanner，将文本切分为 tokens，每个 tokens 会有 name 和 value，其中 name 在 parser 中用于匹配预定义的规则（parser.y），匹配规则时，不断的从 scanner 中获取 token，当能完整匹配上一条规则时，会将匹配上的 tokens 替换为一个新的变量。同时，在每条规则匹配成功后，可以用 tokens 的 value，构造 ast 中的节点或者是 subtree。对于 builtin 函数来说，一般的形式为 name(args)，scanner 中要识别 function 的 name，括号，参数等元素，parser 中匹配预定义的规则，构造出一个 ast 的 node，这个 node 中包含函数参数、函数求值的方法，用于后续的求值。

#### 求值

求值过程是根据输入的参数，以及运行时环境，求出函数或者表达式的值。求值的控制逻辑在optimizer/evaluator/evaluator.go 中。对于大部分 builtin 函数，在 parse 过程中被解析为FuncCallExpr，求值时，通过 FnName 在 builtin.Funcs 表中（expression/builtin/builtin.go）找到对应的函数实现，然后将函数参数以及环境（buitlin.ExprEvalCtx）设置好，最后调用求值函数。

### **整体流程**

* 修改 parser/scanner.l 以及 parser/parser.y

    * 在 scanner.l 中增加规则，将文本解析为 token（对于 builtin 函数，主要是解析函数名）

    * 在 parser.y 中增加规则，将token序列转换成 ast 的 node

    * 在 parser_test.go 中，增加 parser 的单元测试

    * make

* 修改 expression/builtin/builtin.go

    * 实现该函数的功能，并将其 name 和实现注册到 builtin.Funcs 中

* 在 expression/builtin/ 目录下，实现该函数的求值方法

    * 通过函数参数以及运行时环境进行求值

* 写单元测试

    * 在 builtin 目录下，为函数的实现增加单元测试

### **示例**

这里以新增 connection_id() 支持的 [PR](https://github.com/pingcap/tidb/pull/718) 为例

首先看 parser/scanner.l：

> connection_id	{c}{o}{n}{n}{e}{c}{t}{i}{o}{n}_{i}{d} 

这里是定义了一个规则，当发现文本是 「connection_id」 时，转换成一个变量，名称为 connection_id，其中 {c} 的意思是引用了一个叫c的变量，这个变量在前面定义，其规则为 c [c|C] ，意思是大写或者小写的 「c」 都识别为一个叫 c 的变量。其他的字符同理。这样 connection_id 的字面值，不管是大写、小写还是大小写混合情况，都可以处理。
```go
{connection_id}		lval.item = string(l.val)
                    return connectionID
```

这两行的意思是，对于 connection_id 变量，scanner 返回的 token name 是 connectionID ( parser 可以引用这个名字)，其 value 是这段文本的字面值。当parser中只需要变量名，不需要变量值时，可以省略第一行中的 lval.item = string(l.val)。一般情况下，当解析 MySQL 的[保留关键词](http://dev.mysql.com/doc/refman/5.7/en/keywords.html)时，我们可以只需要变量名，不要变量值，原因是这些关键词不能作为 SQL 语句中的identifier，其字面值多半在 parse中不需要。

比如对于 conNection_ID() 这样一个文本，我们可以在 parser 中获取一个名字叫 connectionID 的 token，其值为「conNection_ID」，还有 ( 和 ) 这两个 token。

再看parser/parser.y:

> connectionID   "CONNECTION_ID"

这行的意思是从 scanner 中拿到 connectionID 这个 token 后，我们给他起个名字叫 "CONNECTION_ID”，下面的规则匹配时，我们都使用这个名字。

由于 connection_id 不是 MySQL 的关键字，我们把规则放在 FunctionCallNonKeyword 下，

> "CONNECTION_ID" '(' ')' 
> {
>      $$ = &ast.FuncCallExpr{FnName: model.NewCIStr($1.(string))}
> }

这里的意思是，当 scanner 输出的 token 序列满足这种 pattern 时，我们将这些 tokens 规约为一个新的变量，叫 FunctionCallNonKeyword （通过给$$变量赋值，即可给FunctionCallNonKeyword 赋值），也就是一个 AST 中的 node，类型为 *ast.FuncCallExpr。其成员变量 FnName 的值为 $1 的内容，也就是规则中第一个 token 的 value。

至此我们已经成功的将文本 "connection_id()” 转换成为一个 AST node，其成员 FnName 记录了函数名 ”connection_id”，用于后面的求值。

如果想引用这个规则中某个 token 的值，可以用 $x 这种方式，其中 x 为 token 在规则中的位置，如上面的规则中，$1为 "CONNECTION_ID”，$2为 ’(’ ， $3 为 ’)’ 。$1.(string) 的意思是引用第一个位置上的 token 的值，并断言其值为 string 类型。

如果函数还需要参数，可以在规则中增加参数的规则，以 upper 函数为例：

```go
"UPPER" '(' Expression ')'
{
    $$ = &ast.FuncCallExpr{FnName: model.NewCIStr($1.(string)), Args: []ast.ExprNode{$3.(ast.ExprNode)}} // 解析的参数放在Args成员变量中
}
```

函数注册在builtin.go中的Funcs表中：

> "connection_id": {builtinConnectionID, 0, 0, true, false},

builtinConnectionID：该 builtin 函数的实现在 builtinConnectionID 这个函数中

0：最少的参数个数

0：最多的参数个数，语法parse过程中，会检查参数的个数是否合法

true: 该函数是静态函数

false: 该函数不是聚合函数 （sum/avg/count 这种是聚合函数）

函数实现在 info.go 中，一些细节可以看下面的代码以及注释

```go
func builtinConnectionID(args []interface{}, data map[interface{}]interface{}) (v interface{}, err error) { //返回值必须是这两个
	c, ok := data[ExprEvalArgCtx]
	if !ok {
		return nil, errors.Errorf("Missing ExprEvalArgCtx when evalue builtin")
	}
	ctx := c.(context.Context) // 执行环境，可以用于在整个语句的执行过程中传递数据
	idValue := ctx.Value(ConnectionIDKey) // 为了解决循环依赖，context的接口设计为SetValue和Value，分别用于绑定和获取变量
	if idValue == nil {
		return nil, terror.MissConnectionID
	}
	id, ok := idValue.(int64)
	if !ok {
		return nil, terror.MissConnectionID.Gen("connection id is not int64 but %T", idValue)
	}
	return id, nil
}
```
