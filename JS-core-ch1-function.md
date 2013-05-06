# 1. 函數（Function）

+ JavaScript 的函數是第一級物件。

+ C、Java 或類似語言是以 `{ }` 來提供作用域，JavaScript 則是以 `function` 區塊提供作用域。

## 1.2 定義函數

## 1.3 調用函數

###	1.3.1 函數調用

###	1.3.2 方法調用

###	1.3.3 建構式調用

###	1.3.4 間接調用

## 1.4 函數的種類

[ECMAScript][ECMAScript] 標準中包含了三種類型的函數，各有自己的特性。

### 1.4.1 函數宣告（Function Declaration）

函數宣告是指這樣的函數

+ 有函數名

+ 代碼位置在：要麼在程式級別或者直接在另外一個函數的函數體（Function Body）中

+ 在進入函數上下文時創建的

+ 會影響變數物件

函數宣告是以如下形式聲明的：

```javascript
function exampleFunc() { ... }  //要注意的是，函數宣告的句尾無需加(;)
```

這類函數的主要特性是：

+ 只有它們可以影響變數物件（存儲在上下文的 variable object 中）。

+ 它們在執行代碼階段就已經存在了（因為函數宣告在進入上下文階段就收集到了 variable object 中）。

+ 函數宣告在代碼中的位置：

	```javascript
	function globalFunc() {     // 函數宣告可以直接在全域上下文中
	    function innerFunc() {} // 或者在另外一個函數的函數體中
	}
	```

	除了上述提到了兩個位置，其他位置均不能出現函數宣告。

### 1.4.2 函數表達式（Function Expression）

函數表達式是指這樣的函數：

+ 代碼位置必須是在表達式的位置

+ 函數名字可有可無

+ 不會影響變數物件

+ 在執行代碼階段時創建

這類函數的主要特性是：

+ 它們的程式碼總是在表達式的位置

	```javascript
	var foo = function () { ... };  //最簡單的表達式的例子，就是賦值表達式
	```

上述程式碼將一個匿名函數表達式賦值給了變數 `foo`，之後該函數就可以通過 `foo` 被訪問了。

函數表達式也可以有名字：

```javascript
var FOO = function foo() { ... };
```

有名稱的函數表達式優點是，在函數表達式的外部可以通過變數 `FOO` 調用函數，而在函數內部（比如遞迴調用），還可以用 `foo` 來調用函數。 **要注意的是在 `FOO` 函數外部是無法調用 `foo` 的。**

__區分函數宣告與有名字的函數表達式__

當函數表達式有名字的時候，它很難與函數宣告區分。不過，還是可以從定義來區分， **因為函數表達式總是出現在表達式的位置** 。

```javascript
var foo = function () { ... };  // 等於後面只能是表達式
(function foo() {...});         // 在括弧中（grouping operator）只可能是表達式
[function foo () {...}];        // 在陣列初始化中也只能是表達式
1, function foo () {...};       // 逗號操作符後只能跟表達式
!function foo () {...};         // 驚嘆號操作符也只能跟表達式
...
```

__匿名函數（Anonymous Function）__

TODO

__有名字的函數表達式（named function expression）的特性__

當函數表達式有名字之後，就產生了一個重要的特性。

正如在定義中提到的，函數表達式是不會影響上下文的變數物件的（這就意味著不論是在定義前還是在定義後，都是不可能通過名字來進行調用的）。

然而，函數表達式可以通過自己的名字進行遞迴呼叫：

```javascript
(function foo(bar) {
  if (bar) { return; }
  foo(true); // "foo" name is available
})();
```

__函數表達式能帶來什麼好處?__

使用函數表達式能避免對變數物件造成“污染”！

最簡單的例子就是將函數作為參數傳遞給另外一個函數

```javascript
function foo(callback) {
  callback();
}

foo(function bar() {
  ...
});

foo(function baz() {
  ...
});
```

若將函數表達式指定給變數，這樣變數就儲存對函數表達式的引用，如此一來，函數就會保留在記憶體中，並在之後可以通過變數來訪問。

```javascript
var foo = function _foo() { console.log("foo"); };
foo();   // output foo
```

__宣告式 v.s. 表達式__

到底該用函數宣告式還是函數表示式呢?

### 1.4.3 立即函數（Immediate Function）

故名思義，立即函數是“創建後就馬上執行的函數”。

__如何創建立即函數__

為什麼常在書中或其它地方看到，創建立即函數都有括弧包著函數？

從標準中來看，創建立即函數的規則是函數必須在表達式的位置並且調用它。

所以立即函數，它必須是函數表達式，而不能是函數宣告。

而創建表達式最簡單的方式就是使用上述提到的組操作符。因為在組操作符中只能是表達式。

__表達式的一些語法限制__

標準中提到，表達式語句（ExpressionStatement）不能以左大括弧 `{` 開始，因為這樣一來就和程式碼區塊衝突了，也不能以 `function` 關鍵字開始，因為這樣又和函數宣告衝突了。

```javascript
function () { ... }();
function foo() { ... }();
```

上述兩段程式碼都會拋出錯誤，只是原因不同。

+ 第一個例子中，直譯器會以函數宣告來處理，因為它看到了是以 `function` 開始的。既然是個函數宣告，則缺少函數名(一個函數宣告其名字是必須的)。

+ 第二個例子中，看上去已經有了名字（`foo`），應該會正確執行。然而，這裡還是會拋出語法錯誤，因為組操作符內部缺少表達式。這個例子中，函數宣告後面的 `()` 會被當組操作符來處理，而非函數調用的 `()` 。

	```javascript
	function foo(x) { console.log(x); }(1); // 這裡只是組操作符，並非調用
	foo(10); // 這裡是調用, 10
	```

	上述程式碼其實就是以下程式碼

	```javascript
	function foo(x) { console.log(x); } // function declaration
	(1);                                // 含表達式的組操作符
	(function () {});                   // 另外一個組操作符包含一個函數表達式
	```

回到創建立即函數上，如之前所述

> 它必須要是個函數表達式，而不能是函數宣告。

而創建表達式最簡單的方式就是使用上述提到的組操作符，因為在組操作符中只能是表達式。

如此一來直譯器將會以函數表達式的方式來處理，這樣的函數將在執行階段創建出來，然後馬上執行，隨後被移除。

```javascript
(function foo(x) {
	console.log(x);
})(1);               //output 1， 這樣就是函數調用，而不再是組操作符了
```

在下面的例子中，其括弧就不再是必須的了，因為函數本來就在表達式的位置了，直譯器自然會以函數表達式來處理，並且會在執行程式碼階段創建該函數

```javascript
var foo = {
	bar : function (x) {
		return x ;
	}(1)
};
console.log(foo.bar);   // output 1
```

如果要在函數創建後馬上進行函數調用，但函數不在表達式的位置時，括弧就是必須的。
除了使用括弧的方式將函數轉換成為函數表達式之外，還有其他的方式

```javascript
1, function () { ... }();
!function () {... }();
```

當然還有很多方式可以創建立即函數，不過，括弧是最通用也是最優雅的方式。


__同樣是立即函數的宣告，下列程式碼的差異在哪?__

```javascript
(function () {})();
!function () { ... }();
```

__精簡__

對於 JavaScript，精簡是一種很重要的特性，因為在進入頁面後 JavaScript 會馬上被下載。如果 JavaScript 的程式碼能越小，頁面載入的時間也會變快。
比較下列程式碼

```javascript
(function(){})() // 16 characters
!function(){}()  // 15 characters
```

顯然，`!function(){}()` 比 `(function(){})()` 有更高的壓縮率。

__否定的返回值__

```javascript
var a = (function(){})()
var b = !function(){}()
```

上述函數沒有返回值，變數 `a` 的值將會是 `undefined`，由於 `!undefined` 的值為 `true` ，變數 `b` 會被設置成 `true`。

對於想要“否定返回值的函數”或“任何事物都必需返回非 `null` 或非 `undefined` ”的設計者來說，這是一項優點。

1.5 Object 與 Function

JavaScript 中有兩個特殊的物件：`Object` 與 `Function`，它們都可做為建構函數，用於創建物件。
在 JavaScript 中所有的物件都繼承自 `Object` 原型，而 `Function`又充當了物件的建構函數，乍看之下有點難懂，下面用圖示來解示。

1.6 使用 new 關鍵字創建物件過程

TODO

以下部份將用程式碼來解釋物件創建過程

TODO



[ECMAScript]: https://zh.wikipedia.org/wiki/ECMAScript