# **十分钟成为 TiDB Contributor 系列 | 添加內建函数**

最近我们对 TiDB 代码做了些改进，大幅度简化了添加內建函数的流程，这篇教程描述如何为 TiDB 新增 builtin 函数。首先介绍一些必需的背景知识，然后介绍增加 builtin 函数的流程，最后会以一个函数作为示例。

### **背景知识**

SQL 语句发送到 TiDB 后首先会经过 parser，从文本 parse 成为 AST（抽象语法树），通过 Query Optimizer 生成执行计划，得到一个可以执行的 plan，通过执行这个 plan 即可得到结果，这期间会涉及到如何获取 table 中的数据，如何对数据进行过滤、计算、排序、聚合、滤重以及如何对表达式进行求值。
对于一个 builtin 函数，比较重要的是进行语法解析以及如何求值。其中语法解析部分需要了解如何写 yacc 以及如何修改 TiDB 的词法解析器，较为繁琐，我们已经将这部分工作提前做好，大多数 builtin 函数的语法解析工作已经做完。
对 builtin 函数的求值需要在 TiDB 的表达式求值框架下完成，每个 builtin 函数被认为是一个表达式，用一个 ScalarFunction 来表示，每个 builtin 函数通过其函数名以及参数，获取对应的函数类型以及函数签名，然后通过函数签名进行求值。
总体而言，上述流程对于不熟悉 TiDB 的朋友而言比较复杂，我们对这部分做了些工作，将一些流程性、较为繁琐的工作做了统一处理，目前已经将大多数未实现的 buitlin 函数的语法解析以及寻找函数签名的工作完成，但是函数实现部分留空。***换句话说，只要找到留空的函数实现，将其补充完整，即可作为一个 PR。***

### **添加 builtin 函数整体流程**

* 找到未实现的函数
在 TiDB 源码中的 expression 目录下搜索 `errFunctionNotExists`，即可找到所有未实现的函数，从中选择一个感兴趣的函数，比如 SHA2 函数：
```
func (b *builtinSHA2Sig) eval(row []types.Datum) (d types.Datum, err error) {
    return d, errFunctionNotExists.GenByArgs("SHA2")
}
```

* 实现函数签名
接下来要做的事情就是实现 eval 方法，函数的功能请参考 MySQL 文档，具体的实现方法可以参考目前已经实现函数。

* 在 typeinferer 中添加类型推导信息
在 plan/typeinferer.go 中的 handleFuncCallExpr() 里面添加这个函数的返回结果类型，请保持和 MySQL 的结果一致。全部类型定义参见 [MySQL Const](https://github.com/pingcap/tidb/blob/master/mysql/type.go#L17)。
```
    * 注意大多数函数除了需要填写返回值类型之外，还需要获取返回值的长度。
```

* 写单元测试
在 expression 目录下，为函数的实现增加单元测试，同时也要在 plan/typeinferer_test.go 文件中添加 typeinferer 的单元测试

* 运行 make dev，确保所有的 test case 都能跑过

### **示例**

这里以[新增 SHA1() 函数的 PR](https://github.com/pingcap/tidb/pull/2781/files) 为例，进行详细说明
首先看 `expression/builtin_encryption.go`：
将 SHA1() 的求值方法补充完整
```
func (b *builtinSHA1Sig) eval(row []types.Datum) (d types.Datum, err error) {
    // 首先对参数进行求值，这块一般不用修改
    args, err := b.evalArgs(row)
    if err != nil {
        return types.Datum{}, errors.Trace(err)
    }
    // 每个参数的意义请参考 MySQL 文档
    // SHA/SHA1 function only accept 1 parameter
    arg := args[0]
    if arg.IsNull() {
        return d, nil
    }
    // 这里对参数值做了一个类型转换，函数的实现请参考 util/types/datum.go
    bin, err := arg.ToBytes()
    if err != nil {
        return d, errors.Trace(err)
    }
    hasher := sha1.New()
    hasher.Write(bin)
    data := fmt.Sprintf("%x", hasher.Sum(nil))
    // 设置返回值
    d.SetString(data)
    return d, nil
}
```
接下来给函数实现添加单元测试，参见 `expression/builtin_encryption_test.go`：
```
var shaCases = []struct {
    origin interface{}
    crypt  string
 }{
    {"test", "a94a8fe5ccb19ba61c4c0873d391e987982fbbd3"},
    {"c4pt0r", "034923dcabf099fc4c8917c0ab91ffcd4c2578a6"},
    {"pingcap", "73bf9ef43a44f42e2ea2894d62f0917af149a006"},
    {"foobar", "8843d7f92416211de9ebb963ff4ce28125932878"},
    {1024, "128351137a9c47206c4507dcf2e6fbeeca3a9079"},
    {123.45, "22f8b438ad7e89300b51d88684f3f0b9fa1d7a32"},
 }
 
 func (s *testEvaluatorSuite) TestShaEncrypt(c *C) {
    defer testleak.AfterTest(c)() // 监测 goroutine 泄漏的工具，可以直接照搬
    fc := funcs[ast.SHA]
    for _, test := range shaCases {
        in := types.NewDatum(test.origin)
        f, _ := fc.getFunction(datumsToConstants([]types.Datum{in}), s.ctx)
        crypt, err := f.eval(nil)
        c.Assert(err, IsNil)
        res, err := crypt.ToString()
        c.Assert(err, IsNil)
        c.Assert(res, Equals, test.crypt)
    }
    // test NULL input for sha
    var argNull types.Datum
    f, _ := fc.getFunction(datumsToConstants([]types.Datum{argNull}), s.ctx)
    crypt, err := f.eval(nil)
    c.Assert(err, IsNil)
    c.Assert(crypt.IsNull(), IsTrue)
}
* 注意，除了正常 case 之外，最好能添加一些异常的case，如输入值为 nil，或者是多种类型的参数
```
最后还需要添加类型推导信息以及 test case，参见 `plan/typeinferer.go`，`plan/typeinferer_test.go`：
```
case ast.SHA, ast.SHA1:
        tp = types.NewFieldType(mysql.TypeVarString)
        chs = v.defaultCharset
        tp.Flen = 40
```
```
        {`sha1(123)`, mysql.TypeVarString, "utf8"},
        {`sha(123)`, mysql.TypeVarString, "utf8"},
```


编辑按：添加 TiDB Robot 微信，加入 TiDB Contributor Club，无门槛参与开源项目，改变世界从这里开始吧（萌萌哒）。

![TiDB Robot](tidb_bot.png)