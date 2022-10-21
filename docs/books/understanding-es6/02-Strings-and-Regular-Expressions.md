# 字符串和正则表达式

## 更好的 Unicode 支持

在 `ESCMAScript 6` 之前，`JavaScript` 字符串一直基于 16 位字符编码（UTF - 16） 进行构建。每 16 位的序列是一个编码单元（code unit），代表一个字符。过去 16 位足以包含任何字符，直到 Unicode 引入扩展字符集，编码规则才不得不进行更改。

### UTF-16 码位

`Unicode ` 的目标是为全世界每一个字符提供全球唯一标识符。"全球唯一的标识符” 也被称作**码位** （code point），是从 0 开始的数值。而表示字符的这些数值和码位，我们称之为字符编码 （character encode）。字符编码必须将码位编码为内部一致的编码单元。对 UTF - 16 来说，码位可以由多种编码单元表示。

在 `UTF-16` 中，前 `2^16` 个码位均以 16 位的编码单元表示，这个范围被称作**基本多文种平面（BMP，Basic Multilingual Plane）** 。超出这个范围的码位则要归属于某个辅助平面（supplementary plane）, 其中的码位仅用 16 位就无法表示了。为此，`UTF-16 ` 引入了**代理对（surrogate pair）**，其规定用两个 16 位编码单元表示一个码位。也就是说，**字符串里的字符有两种，一种是由一个编码单元 16 位表示的 BMP（基本多文种平面）字符，另一种是由两个编码单元 32 位表示的辅助平面字符**。

#### ECMAScript5 中对 UTF16 编码字符的处理

在 `ECMAScript5` 中，所有字符串的操作都是基于 16 位编码单元。如果采用同样的方式处理包含代理的 `UTF-16` 编码字符，得到的结果可能与预期不符：

```JavaScript
let text = '𠮷'; // 不知道什么字, 日文字符

console.log(text.lengtt); // 2,由2个编码单元 32 位标示
console.log(/^.$/.test(text)); //false, 匹配任意一个字符
console.log(text.charAt(0)); // "", 返回字符串中的指定字符
console.log(text.charAt(1)); // ""
console.log(text.charCodeAt(0)); // 55362
console.log(tezt.charCodeAt(1)); // 57271

```

> String.prototype.charCodeAt() 返回 0 到 65535 之间的整数，表示给定索引处的 UTF-16 代码单元  1、在 Unicode 编码单元表示一个单一的 UTF-16 编码单元的情况下，UTF-16 编码单元匹配 Unicode 编码单元。
>
> 2、但在——例如 Unicode 编码单元 > 0x10000 的这种——不能被一个 UTF-16 编码单元单独表示的情况下，只能匹配 Unicode 代理的第一个编码单元。
>
> 3、

Unicode 字符“𠮷” 是通过代理对来表示的，JavaScript 字符串操作将其视为两个 16 位字符。这就意味着：

* 变量 text 的长度事实上是1，但它的 `length` 属性值却为 2。

* 变量 text 被判定为两个字符，因此匹配单一字符的正则表达式会失效。

  * ```javascript
    /^.{2}$/.test(text); // true
    ```

* 前后两个 16 位的编码单元都不表示任何可打印的字符，因此 `charAt()` 方法不会返回合法的字符串。

* `charCodeAt()` 方法同样不能正确地识别地识别字符。它会返回每个 16 位编码单元对应的数值，在 ECMACScript 5 中，这已经是你可以得到的最接近 text 真实值的结果了。

### codePointAt() 方法

`ECMAScript 6` 新增加了完全支持 UTF-16 的 `codePointAt()`  方法，这个方法接受**编码单元的位置而非字符位置作为参数**，返回与字符串中给定位置对应的码位，即一个整数值。

```javascript
let text = '𠮷a';

console.log(text.charCodeAt(0)); // 55362
console.log(text.charCodeAt(1)); // 57271
console.log(text.charCodeAt(2)); // 97

console.log(text.codePointAt(0)); // 134071 完整的码位，即使这个码位包含多个编码单元
console.log(text.codePointAt(1)); // 57271
console.log(text.codePointAt(2)); // 97
```

对于 BMP 字符集中的字符，`codePointAt()` 方法的返回值与 `charCodeAt()` 方法相同，而对于非 BMP 字符集来说返回值则不同。

**检测一个字符占用的编码单元数量函数**

Unicode 只有两种情况：

1. 16位字符编码

2. 32位字符编码

所以构建这个函数只需要调用字符的 `codePointAt()` 方法，获取当前字符的 Unicode 码，然后判断是否超出边界值：

```javascript
function is32Bit(c) {
  return c.codePointAt(0) > 0xFFFF;
}

console.log(is32Bit('𠮷')); // true
console.log(is32Bit('a')); // false
```

用 16 位表示的字符集上界为十六进制 `FFFF` , 所有超过这个上界的码位一定由两个编码单元来表示，总共有 32 位。

### String.fromCodePoint() 方法

根据指定的码位生成一个字符。

```javascript
let text = '𠮷';

console.log(String.fromCodePoint(text.codePointAt(0))); // '𠮷'
```

此方法可以看成是更完整版的 `String.formCharCode()` 。同样，对于 `BMP` 中的所有字符，这两个方法的执行结果都相同。只有传递非 BMP 的码位作为参数时，二者执行结果才有可能不同。

### normaliza() 方法 （ES6）

ECMAScript 6 为字符串新增了一个 `normaliza()` 方法，它可以提供 `Unicode ` 的标准化形式。因为 `Unicode` 有一个有趣的之处是 "**对不同的字符进行排序或比较操作会存在一种可能，它们是等效的。**"

有两种方式定义这种关系：

1. 规范的等效是指无论从哪个角度来看，两个序列的码位都是没有区别的；
2. 兼容性，两个互相兼容的码位序列看起来不同，但是在特定的情况下它们被互相交换作用使用。例如：字符  "æ" 和含两个字符的字符串 "ae" 可以互换使用，但严格来讲他们不是等效的，需要通过某些方法把这种等效关系标准化。(有待思考🤔)

#### Unicode 标准化形式

* 以标准等价方式分解，然后以标准等价方式重组（"NFC"）,默认选项。
* 以标准等价方式分解（"NFD"）
* 以兼容等价方式分解（"NFKC"）kp
* 以兼容等价方式分解，然后以标准等价方式重组（"NFKD"）

> 重点：总之对比字符串之前，一定先把它们标准化为同一种形式。

```javascript
// 排序先将 values 数组里的所有字符串统一标准化（NFC）
const normalized = values.map(value => value.noramize()); // 其他标准化形式 value.normalize('NFD')
// 排序
normalized.sort((a, b) => a - b);
```

### 正则表达式 u 修饰符

正则表达式对字符串的操作默认将字符串的每个字符按照 16 位编码单元处理。对于超出 16 位编码单元（BMP）的 32 位 **辅助平面字符**，ECMAScript 6 给正则表达式定义了一个支持 Unicode 的 `u` 修饰符。

#### u 修饰符实例

正则表达式添加了 `u` 修饰符时，它就从**编码单元操作模式**转换成**字符模式**。正则表达式就不会以 2 个编码单元（**代理对**）处理方式视为 2 个字符。

```javascript
let text = '𠮷';

console.log(text.length); // 2
console.log(/^.$/.test(text)); // false, 正则匹配一个任意字符，按照默认 text 是2个字符
console.log(/^.$/u.test(text)); // true
```

#### 计算码位数量

ECMAScript 6 目前不支持字符串码位数量的检测（length 属性还是返回字符串编码单元的数量），但是可以通过正则表达式的支持的新修饰符 `u` 来解决这个问题。

```javascript
function codePointLength(text) {
  const result = text.match(/[\s\S]/gu); // 匹配空白和非空白字符 （换行、制表符、空格等）
  return result ? result.length : 0;
}

console.log('𠮷'.length); // 2
console.log(codePointLength('𠮷')); // 1
```

> \s : 匹配任何空白字符，包括空格、制表符、换页符等等。等价于 [ \f\n\r\t\v]。注意 Unicode 正则表达式会匹配全角空格符。
>
> \S : 匹配任何非空白字符。等价于 [^ \f\n\r\t\v]。
>
> 注意：此方法在统计长字符串中的码位数量时，运行效率很低，可以使用字符串迭代器解决效率低的问题。

#### 检测 u 修饰符支持 （Polyfill）

兼容不支持 ES6 的 JavaScript 引擎。

```javascript
function hasRegExpU() {
  try {
    var pattern = new RegExp(".", "u");
    return true;
  } catch(ex) {
    return false;
  }
}
```

利用 `RegExp` 构造函数并传入 `u`  字符串作为参数，老式浏览器引擎支持这个语法，但是如果不支持 `u` 修饰符会抛出错误。

### 字符串扩展方法（ES6）

#### 字符串中的子串识别

* `includes()` 方法，如果在字符串中检测到指定文本则返回 `true` ，否则返回 `false` 。
* `startsWith()` 方法， 如果在字符串的起始部分检测到指定的文本则返回 `true`, 否则返回 `false` 。
* `endsWith()` 方法，如果在字符串的结束部分检测到指定文本则返回 `true`, 否则返回 `false`。

以上 3 个方法都接受两个参数：第一个参数指定要搜索的文本；第二个参数是可选的，指定一个开始搜索位置的索引值。默认是从起始位置开始匹配。

```javascript
const msg = "Hello world!";

console.log(msg.includes('lo')); // true

console.log(msg.startsWith('He')); // true
console.log(msg.startsWith('wo')); // false

console.log(msg.endsWith('d!')); // true
console.log(msg.endsWith('~')); // false

console.log(msg.starsWith('wo', 6)); // true
```

> ⚠️ 注意：
>
> `includes()`、 `startsWith()`、`endsWith()` 这 3 个方法，第一个参数必须是字符串，否则会产生报错。
>
> `indexOf()`、 `lastIndexOf()` 方法传递第一个参数为正则表达式，则会转化为一个字符串并搜索它。

#### repeat() 方法

功能就是字面意思 "重复"，接收一个 number 类型的参数，表示该字符串的重复次数，返回值是当前字符串重复一定次数后的新字符串。

```javascript
console.log('money '.repeat(6)); // money money money money money money
console.log('six '.repeat(6)); // six six six six six six
```

### 其他正则表达式语法变更 （ES6）

#### 正则表达式 y 修饰符

`y` 修饰符是 ECMAScript 6 成为正则表达式的一个专有扩展。当字符串中开始字符匹配时，它会通知搜索从正则表达式的 `lastIndex` 属性开始进行，如果在指定的位置没能成功匹配，则停止继续匹配。

`y` 修饰符 和 `g` 修饰符功能类似，都是全局匹配，后一次匹配都从上一次匹配成功的 `lastIndex` 开始。不同之处在于，`g `修饰符只要剩余位置中存在匹配就可，而 `y` 修饰符确保匹配必须从剩余的第一个位置开始，会影响正则表达式搜索过程中的 `sticky` (粘滞) 属性。

**示例**

```javascript

let text = 'hello1 hello2 hello3',
  pattern = /hello\d\s?/,
  result = pattern.exec(text),
  globalPattern = /hello\d\s?/g, // g 修饰符
  globalResult = globalPattern.exec(text),
  stickyPattern = /hello\d\s?/y, // y 修饰符
  stickyResult = stickyPattern.exec(text);

// 匹配值
console.log(result[0]); // "hello1 "
console.log(globalResult[0]); // "hello1 "
console.log(stickyResult[0]); // "hello1 "

// exec 执行一次正则匹配后，正则表达式的 lastIndex 属性
console.log(pattern.lastIndex); // 0
console.log(globalPattern.lastIndex); // 7
console.log(stickyPattern.lastIndex); // 7

// 手动修改下正则表达式的 lastIndex 属性，测试一下 sticky 属性影响
// lastIndex 为 1，所以从第二个字符 “e” 开始匹配
pattern.lastIndex = 1; // 没有修饰符，每次匹配都是从头开始，lastIndex 直接被忽略
globalPattern.lastIndex = 1;
stickyPattern.lastIndex = 1;

result = pattern.exec(text);
globalResult = globalPattern.exec(text);
stickyResult = stickyPattern.exec(text); // 使用了 y 修饰符的粘滞

console.log(result[0]); // "hello1 "
console.log(globalResult[0]); // "hello2 "
console.log(stickyResult[0]); // 抛出错误： Cannot read property '0' of null

// 报错中断后，再次的执行下面代码
stickyResult = stickyPattern.exec(text);
console.log(stickyResult[0]); // "hello1 "
```

`y` 修饰符会把上次匹配后面一个字符的索引保存在 `lastIndex` 中；如果该操作匹配的结果为空，则 `lastIndex` 会被重置为 `0` 。`g` 修饰符的行为与此相同。

##### 注意事项

**y 修饰符**

1. 只有调用 `exec()` 和 `test()` 这些正则表达式对象的方法时才会涉及 `lastIndex` 属性；调用字符串的方法，例如 `macth()` 则不会触发粘滞行为。
2. 对于粘滞正则表达式而言，如果使用 `^` 字符 来匹配字符串开端，只会从字符串的起始位置或多行模式的首行进行匹配。当 `lastIndex` 的值为 0 时，如果正则表达式中含有 `^` ，则是否使用粘滞正则表达式并无差别；如果 `lastIndex` 的值在单行模式下不为 0，或在多行模式下不对应行首，则粘滞的表达式永远不会匹配成功。

##### 检测 y 修饰符是否存在

通过属性名来进行检测， 检测正则表达式 `sticky`  粘滞属性是否支持。

```javascript
const pattern = /hello\d/y;
console.log(pattern.sticky); // true
```

如果浏览器引擎支持粘滞修饰符，则 `sticky` 属性值为 `true`, 否则为 `false`。这个是只读属性。

##### 检测引擎对它的支持程度

检测浏览器引擎对 `y` 修饰符和  `u` 修饰符的支持程度，两种方式都是通过 `RegExp` 构造函数来进行（规避语法错误）：

```JavaScript
function hasRegExpY() {
  try {
    var parrent = new RegExp(".", "y");
    return true;
  } catch (ex) {
    return false;
  }
}
```

#### 正则表达式的复制

对于正则表达式的复制，就是通过 `RegExp` 的构造函数传递正则表达式作为参数来复制这个正则表达式。

这种方式的复制在 `ECMAScript 5` 和 `ECMAScript 6` 有一个区别：

* 在 `ECMAScript 5` 传递第一个参数为正则表达式时，不能传递第二个参数为正则表达式指定一个修饰符，否则会抛出一个错误
* `ECMAScript 6` 环境下，正常运行，也支持了传递第一个参数为正则表达式时，也可以传递第二个参数指定修饰符。

```javascript
// 复制方式
var re1 = /ab/i,
    re2 = new RegExp(re1);

// ES5 中会抛出错误, ES6 支持
re3 = new RegExp(re1, 'g');
console.log(re3.toString()); // "/ab/g"
```

#### flags 属性（ES6）

获取正则表达式的修饰符。

ES5 的时候获取正则表达式修饰，只能通过 `toString` 的方法来进行：

```javascript
function getFlags (reg) {
  var text = reg.toString();
  return text.substring(text.lastIndexOf('/') + 1, text.length);
}

// 方式二（替换截取方法）
function getFlags(reg) {
  var text = reg.toString();
  return text.slice(text.lastIndexOf('/') + 1);
}

const pattern = /hello\d/y;
console.log(pattern.source) // "hello\d"
console.log(getFlags(pattern)); // "y"
```

ES6 新增 **flags** 属性：

```javascript
const pattern = /hello\d/y;
console.log(pattern.flags); // "y"
```

### 模板字符串（模板字面量）

**ECMAScript 6** 模板字面量语法糖支持创建领域专用语言（DSL）,扩展 `ECMAScript` 基础语法糖，其提供一套生成、查询并操作来自其他语言里内容的 DSL，且可以免受注入攻击，例如，XSS、SQL 注入等。

> **领域特定语言**（英语：domain-specific language、DSL）
>
> 指的是专注于某个[应用程序](https://baike.baidu.com/item/应用程序/5985445)领域的[计算机语言](https://baike.baidu.com/item/计算机语言/4456504)。又译作**领域专用语言**。不同于普通的跨领域通用计算机语言(GPL)，领域特定语言只用在某些特定的领域。 比如用来显示网页的[HTML](https://baike.baidu.com/item/HTML)，以及[Emacs](https://baike.baidu.com/item/Emacs)所使用的Emac LISP语言。

相对于 **ECMAScript 5** ，模板字面量支持了：

* 多行字符串
* 基本的字符串格式化 （将变量的值嵌入字符串的能力）
* HTML 转义

```javascript
const firstName = `Junting`;
const lastName = 'Liu';
const str = `My first name is ${firstName},
last name is ${lastName}.`;

console.log(str);
// My first name is Junting,
// last name is Liu.
```

#### 标签模板

```javascript
let message = tag`hello world!`;
```

`tag` 就是标签模板里的**模板标签**。每个模板标签都可以执行模板字面量上的转换并返回最终的字符串。

标签可以是一个函数，调用时传入加工过的模板字面量各个部分数据，但必须结合每个部分来创建结果。

* 第一参数是一个数组，包含 Javascript 解释过后的字面量字符串
* 后面的每一个参数都是每一个占位符的解释值。

```javascript
function tag (literals, ...substitutions) {
  // 返回一个字符串
}
```

> **literals** 第一个参数永远是一个空字符串，确保字符串始端。
>
> **substitutions** 参数的数量永远也比 **literals** 少一个。`substitutions.length === literals.length - 1` 结果总为 `true`

**示例：实现一个简单的模板字面量转换默认行为的函数**

```javascript
const count = 10,
      price = 2.5,
      message = passthru`${count} items cost $${(count * price).toFixed(2)}.`;

function passthru (literals, ...substitutions) {
  console.log(literals); // ["", " items cost $", ".", raw: Array(3)]
  console.log(substitutions); // [10, "25.00"]
  let result = '';

  for (let i = 0; i < substitutions.length; i++) {
    result += literals[i];
    result += substitutions[i];
  }
  result += literals[literals.length - 1];
  return result;
}

console.log(message); // "10 items cost $25.00."
```

标签模板的标签( Tag ) 的职责就是处理字符串最后怎样返回结果，由标签来定。

#### 获取模板字面量的原始值

`String.raw()` 提取原生字符串信息，提取转义或转换前的原生字符串。

```javascript
let message1 = `Multiline\nstring`,
    message2 = String.raw`Multiline\nstring`;

console.log(message1); // "Multiline"
											 // "string"
console.log(message2); // "Multiline\nstring" (Browser)
											 // "Multiline\\nstring" (Node)
```

> 上个示例中，标签函数的第一个参数数组还有一个额外的属性 **raw**, 是一个包含每一个字面值的原生等价信息的数组。
