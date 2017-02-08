本文档用于描述如何为 TiDB 新增 builtin 函数。首先介绍一些必需的背景知识，然后介绍增加 builtin 函数的流程，最后会以一个函数作为示例。

### **背景知识**

SQL 语句在 TiDB 中是如何执行的?

SQL 语句首先会经过 parser，从文本 parse 成为 AST（抽象语法树），通过 optimizer 生成执行计划，得到一个可以执行的 plan，通过执行这个 plan 即可得到结果，这期间会涉及到如何获取 table 中的数据，如何对数据进行过滤、计算、排序、聚合、滤重等操作。对于一个 builtin 函数，比较重要的是 parse 和如何求值。这里着重说这两部分。

#### Parse

TiDB 语法解析的代码在 parser 目录下，主要涉及 misc.go 和 parser.y 两个文件。在 TiDB 项目中运行 make parser 会通过 goyacc 将 parser.y 其转换为 parser.go 代码文件。转换后的 go 代码，可以被其他的 go 代码调用，执行 parse 操作。

将 sql 语句从文本 parse 成结构化 AST 有如下过程。首先是通过 scanner 将文本切分为 tokens，每个 token 会有 name 和 value，其中 name 在 parser 中用于匹配预定义的规则（规则在 parser.y 中定义）。匹配规则时，parser 不断的从 scanner 中获取 token (通过调用 `Scanner.Lex` 方法)，这一过程称为**移进**（shift）；当 parser 发现能完整匹配上一条规则时，会将匹配上的 tokens 替换为一个新的变量，这一过程称为**归约**（reduce）。同时，在每条规则匹配成功后，可以用 tokens 的 value，构造ast 中的节点或者是 subtree。对于 builtin 函数来说，一般的形式为 name(args)，scanner 中要识别 function 的 name、括号、参数等元素；对于匹配预定义规则输入，parser 会构造出一个 ast 的 node，这个 node 中包含函数参数、函数求值的方法，用于后续的求值。

#### 求值

求值过程是根据输入的参数，以及运行时环境，求出函数或者表达式的值。求值的控制逻辑 expression 包中。对于大部分 builtin 函数，在 parse 过程中被解析为 `ast.FuncCallExpr`，求值时首先将 `FuncCallExpr` 转换成 `expression.ScalarFunction`，这时会调用 `NewFunction` 方法 (expression/scalar_function.go)，通过 `FuncCallExpr.FnName` 在 `funcs` 表（expression/builtin.go）中找到对应的函数实现类（`functionClass`），并通过 `functionClass.getFunction` 获得函数实现（`builtinFunc`），最后在对 `ScalarFunction` 求值时会调用 `builtinFunc.eval` 完成函数求值。

### **添加 builtin 函数整体流程**

1. 修改 parser/misc.go ，在 `tokenMap` 中添加函数名到 token code 的映射，其中 token code 是生成的变量，在 parser/parser.y 中通过 %token 声明，由 goyacc 自动生成；
2. 修改 parser/parser.y ：
   * 用 %token 声明函数
   * 根据函数名性质 (请查阅 [mysql 文档](https://dev.mysql.com/doc/refman/5.7/en/keywords.html))，将其添加到 UnReservedKeyword 或 ReservedKeyword 或 NotKeywordToken 规则中
   * 在合适位置添加函数解析规则
3. 在 parser_test.go 中，添加对应函数的单元测试；
4. 修改 ast/functions.go ，定义相应函数名常量，供后续代码引用；
5. 在 expression 包中实现函数，注意函数实现按函数类别分了几个文件，比如时间相关的函数在 expression/builtin\_time.go，函数实现简要说明如下：
   * 定义相应函数类（`functionClass`），实现 `getFunction` 方法
   * 定义函数签名，其应实现 `builtinFunc` 接口，通过实现 `eval` 方法来完成函数逻辑
   * 在 expression/builtin_xxx_test.go 中添加对应函数的单元测试
   * 将函数名及其实现类注册到 `builtin.funcs` 中，这里函数名用到了第4步定义的常量
6. 在 typeinferer 中添加类型推导信息，请保持函数结果类型和 MySQL 的结果一致，全部类型定义参见 [MySQL Const](https://github.com/pingcap/tidb/blob/master/mysql/type.go#L17)
   * 在 plan/typeinferer.go 中的 `handleFuncCallExpr` 里面添加这个函数的返回结果类型
   * 在 plan/typeinferer_test.go 中添加相应测试
7. 运行 `make dev`，确保所有的 test case 都能跑过

### **示例**

这里以新增 [UTC\_TIMESTAMP](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_utc-timestamp) 支持的 [PR](https://github.com/pingcap/tidb/pull/2592) 为例，进行详细说明

1. 首先看 [parser/misc.go](https://github.com/pingcap/tidb/pull/2592/files#diff-2680bef19a08b7dc1c3a74194be5d0f6)：

    在 `tokenMap` 中添加一个 entry
    ```go
    var tokenMap = map[string]int{
        ...
        "UTC_TIMESTAMP":       utcTimestamp,
        ...
    }
    ```

    这里是定义了一个从文本 'UTC\_TIMESTAMP' 到 token code 的映射关系，token code 的由常量 `utcTimestamp` 指定，其值是 goyacc 自动生成的。SQL 对大小不敏感，`tokenMap` 里面统一用大写。

    对于 `tokenMap` 这张表里面的文本，不要被当作 identifier，而是作为一个特别的token。接下来在 parser 规则中，需要对这个 token 进行特殊处理。

2. 看 [parser/parser.y](https://github.com/pingcap/tidb/pull/2592/files#diff-5dbf5cc474f4f1eed5fa9e9796760002):

    ```
    %token  <ident>
            ...
            utcTimestamp    "UTC_TIMESTAMP"
            ...
    ```

    这行的意思是，声明一个叫 `utcTimestamp` 的 token，在 parser 调用 lexer 的 `Lex` 方法时，可能会返回这个 token code，供 parser 识别。此外，我们还给他起了个别名叫 "UTC\_TIMESTAMP"。有 yacc/bison 使用经验的同学可能会好奇为何 `utcTimestamp` 后还跟了字符串，这是 goyacc 支持的语法，名为 literal string，可以在后续规则中替代 `utcTimestamp` 使用，具体说明可以参见[该文档](https://godoc.org/github.com/cznic/y#hdr-LiteralString_field)。

    这里的 `utcTimestamp` 就是 `tokenMap` 里面的那个 `utcTimestamp`，当 parser.y 生成 parser.go 的时，将会生成一个名为 被赋予一个名为 `utcTimestamp` 的 int 常量，即 token code。

    在查阅[文档](https://dev.mysql.com/doc/refman/5.7/en/keywords.html)后得知 UTC_TIMESTAMP 是 MySQL 的保留字，因此我们将其加到 `ReservedKeyword` 规则下。最后，添加该函数解析规则，由于其不是关键字，因此在 `FunctionCallNonKeyword` 下添加如下规则：
    ```
    |   "UTC_TIMESTAMP" FuncDatetimePrec
        {
            args := []ast.ExprNode{}
            if $2 != nil {
                args = append(args, $2.(ast.ExprNode))
            }
            $$ = &ast.FuncCallExpr{FnName: model.NewCIStr($1), Args: args}
        }
    ```

    这里的意思是，当 scanner 输出的 token 序列满足这种 pattern 时，我们将这些 tokens 规约为一个新的变量，叫 `FunctionCallNonKeyword` （通过给$$变量赋值，即可给 `FunctionCallNonKeyword` 赋值），也就是一个 AST 中的 node，类型为 *ast.FuncCallExpr。其成员变量 FnName 的值为 `model.NewCIStr($1)`，其中 `$1` 在生成代码中将被替换成第一个 token （也就是 `utcTimestamp`）的 value。值得一提的是，这里使用了 utcTimestamp 这个 token 的 literal string，倒不是说 token 的 value 就是 "UTC_TIMESTAMP" 这个字符串，其真实的字由 lexer 决定，保存在 `ident string` 这个 union 字段下（因为我们之前声明 `utcTimestamp` 的‘类型’为 `<ident>`）。

    至此我们的parser已经能成功的将文本 "utc\_timestamp()" 转换成为一个 AST node，其成员 `FnName` 记录了函数名 "utc\_timestamp"，我们可以通过这个函数名在后面的 `funcs` 找到函数具体的实现类。

    最后在补充一下yacc规则的基础知识：如果想要在规则处理代码中引用这个规则中某个 token 的值，可以用 $x 这种方式，其中 x 为 token 在规则中的位置，如上面的规则中，$1 为 utcTimestamp，$2 为 FuncDatetimePrec 。$2.(ast.ExprNode) 的意思是引用第2个位置上的 token 的值，并断言其值为 `ast.ExprNode` 类型。

3. 在 [parser/parser_test.go](https://github.com/pingcap/tidb/pull/2592/files#diff-27c45ca411f005e1b8796b12fb53e26c) 添加测试代码

    以上步骤完成后，如果没有文法冲突，你应该可以通过 `make parser` 生成 parser.go 了，快进入 parser 目录执行 `go test` 测试你的成果吧！

4. 参考 [ast/functions.go](https://github.com/pingcap/tidb/pull/2592/files#diff-ade136ede78b393a9e9538c6b7008e02) 为函数名定义一个常量

5. 在 expression 完成对函数逻辑的代码实现：

    在 [expression/builtin_time.go](https://github.com/pingcap/tidb/pull/2592/files#diff-d61eef12d314ca7514bc1960312ba5e4) 中实现函数相关类型，实现代码大致如下：

    ```go
    type utcTimestampFunctionClass struct {
        baseFunctionClass
    }

    func (c *utcTimestampFunctionClass) getFunction(args []Expression, ctx context.Context) (builtinFunc, error) {
        return &builtinUTCTimestampSig{newBaseBuiltinFunc(args, ctx)}, errors.Trace(c.verifyArgs(args))
    }

    type builtinUTCTimestampSig struct {
        baseBuiltinFunc
    }

    // See https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_utc-timestamp
    func (b *builtinUTCTimestampSig) eval(row []types.Datum) (d types.Datum, err error) {
        args, err := b.evalArgs(row)
        if err != nil {
            return types.Datum{}, errors.Trace(err)
        }

        fsp := 0
        sc := b.ctx.GetSessionVars().StmtCtx
        if len(args) == 1 && !args[0].IsNull() {
            if fsp, err = checkFsp(sc, args[0]); err != nil {
                return d, errors.Trace(err)
            }
        }

        t, err := convertTimeToMysqlTime(time.Now().UTC(), fsp)
        if err != nil {
            return d, errors.Trace(err)
        }

        d.SetMysqlTime(t)
        return d, nil
    }
    ```

    其中，`utcTimestampFunctionClass` 实现了 `functionClass` 接口，`builtinUTCTimestampSig` 实现了 `builtinFunc` 接口，其求值过程在上文背景知识中已经介绍过了，在此不再赘述。

    实现了函数后，需要将其注册到 [expression/builtin.go](https://github.com/pingcap/tidb/pull/2592/files#diff-cdbd511e3e5f2bbe0a3c5c173d3938c2) 中的 `funcs` 表里，代码如下：
    ```go
    ast.UTCTimestamp:     &utcTimestampFunctionClass{baseFunctionClass{ast.UnixTimestamp, 0, 1}},
    ```

    意思是，我们可以通过 `ast.UTCTimestamp` 这个函数名找到函数的具体实现，其实现是一个 `functionClass`，也就是我们刚才实现的。构造这个 `functionClass` 时，我们传入了一函数基类，其参数含义是：这个函数的函数名是 `ast.UTCTimestamp`，它接受0到1个参数。我们也可能需要实现一个能接受任意多个参数的函数，此时可将 `baseFunctionClass` 最后一个字段设为-1。

    测试很重要，赶快在 [expression/builtin_time_test.go](https://github.com/pingcap/tidb/pull/2592/files#diff-9efa9861dfe0962aeeda87c63deb0f0f) 添加测试吧！

6. 在 [plan/typeinferer.go](https://github.com/pingcap/tidb/pull/2592/files#diff-686e374c1af6ac094dcaed1db4f0fd44) 实现对函数结果的类型推导，代码如下：

    ```go
    case "now", "sysdate", "current_timestamp", "utc_timestamp":
          tp = types.NewFieldType(mysql.TypeDatetime)
          tp.Decimal = v.getFsp(x)
    ```
    
    意思是这个函数将返回一个 'DATETIME' 类型的结果。如果不确定函数返回类型，一个小窍门/笨办法是：通过 mysql workbench 连接 mysql 数据库，执行 `select utc_timestamp` 看看 :)
    
    目前，函数类型推导和函数实现还是分开的，略有不便，关于这点 pingcap 的小伙伴正在积极改善中，敬请期待！不过当务之急，还是先在添加 [plan/typeinferer_test.go](https://github.com/pingcap/tidb/pull/2592/files#diff-6425b785337ec89d2604eb16f63caac6) 添加测试吧！

7. 至此，一个 builtin 函数已经大功告成，运行 `make dev` 通过所以测试，就可以向 [TiDB](https://github.com/pingcap/tidb) 提 PR 了！
