## 关于Angular表达式中的问号
今天在angular中发现这样的
```
[inputValue] = "values?.listArray"
```
或者这样的
```
<h2>{{course?.description}}</h2>
```
查了一下,问号是可选链接运算符,相当于:
```
values != null ? values.listArray: null
```
不过tyepscript中问号的作用不止于此，分别是:
.  三元运算符
```
// 当 isNumber(input) 为 True 是返回 ? : 之间的部分； isNumber(input) 为 False 时
// 返回 : ; 之间的部分
const a = isNumber(input) ? input : String(input);
参数
// 这里的 ？表示这个参数 field 是一个可选参数
function getUser(user: string, field?: string) {
}
```
. 成员
```
// 这里的？表示这个name属性有可能不存在
class A {
  name?: string
}

interface B {
  name?: string
}
```
. 安全链式调用
```
// 这里 Error对象定义的stack是可选参数，如果这样写的话编译器会提示
// 出错 TS2532: Object is possibly 'undefined'.
return new Error().stack.split('\n');

// 我们可以添加?操作符，当stack属性存在时，调用 stack.split。若stack不存在，则返回空
return new Error().stack?.split('\n');

// 以上代码等同以下代码
const err = new Error();
return err.stack && err.stack.split('\n');
``
person?.name?.firstName 可以理解成 person && person.name && person.firstName,在试图访问的属性后面加上小问号就是 if not undefined 的逻辑

## typescript中除了问号，还有个感叹号用处也很多:
. 一元运算符
```
// ! 就是将之后的结果取反，比如：
// 当 isNumber(input) 为 True 时返回 False； isNumber(input) 为 False 时返回True
const a = !isNumber(input);
```
. 成员
```
// 因为接口B里面name被定义为可空的值，但是实际情况是不为空的，那么我们就可以
// 通过在class里面使用！，重新强调了name这个不为空值
class A implemented B {
  name!: string
}

interface B {
  name?: string
}
```
. 强制链式调用
```
// 这里 Error对象定义的stack是可选参数，如果这样写的话编译器会提示
// 出错 TS2532: Object is possibly 'undefined'.
new Error().stack.split('\n');

// 我们确信这个字段100%出现，那么就可以添加！，强调这个字段一定存在
new Error().stack!.split('\n');
```
