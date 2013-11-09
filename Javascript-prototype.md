# JavaScript 的 `constructor` 、 `prototype` 和 `__proto__` 屬性

## 1. 每一個物件都有 `__proto__` 屬性。

`__proto__` （內部原型）屬性的值就是它的原型對象。當一個物件被創建時,它的 `__proto__` 屬性和內部屬性 [[Prototype]] 指向了相同的物件 （也就是它的構造函數的prototype屬性）。

自從 `__proto__` 出現在 Firefox 中以後,它就變的越來越流行,現在 V8 （Chrome, Node.js）和 Nitro （Safari）也已經支持了它。但 ECMAScript 5 並沒有標准化 `__proto__` ，不過由於它現在的流行程度，它將會成为 ECMAScript 6 規範的一部分。

## 2. 物件的 `constructor` 屬性。
物件的 `constructor` 屬性來自於原型鍊 `__proto__.construcotr`

```javascript
Function A(){}
var a = new A();
console.log(a.constructor); // output Function A(){}
```

實例 `a` 本身沒有 `constructor` 屬性
```jaascript
console.log(a.hasOwnProperty('constructor')); //false
```
所以 `a.constructor` 是透過原型鍊查找到的
```jaascript
console.log(a.constructor === a.__proto__.constructor);  //true
```

## 3. 誰才有 `constructor` 屬性？

函式的 **原型物件** 才有 `constructor` 屬性。
當宣告一個函數的時候，會同時創建一個**原型物件**，賦值到函數的 `prototype` 屬性，作為使用 `new` 生成實例物件的預設原型物件。
該預設 **原型物件** 的內容是：

```javascript
{
    __proto__: Object.prototype,
    constructor:  //指向函數本身
    ...
}
```

`__proto__` 指向 `Object.prototype` 的目的是為了使生成的實例物件繼承頂層物件 `Object.prototype` ;
原型物件中 `constructor` 指向構造函數本身。目的是為了使生成的實例物件 `newObject` 可以直接通過 `newObject.constructor `訪問到構造函數。
同時構造函數和原型物件可以互相訪問也是個良好的設計。

範例：
```javascript
var A = function(){};  
var a = new A();

console.log(a.constructor === A);  //true
```

實例 `a` 的 `constructor` 就是指向 `A`。
而 `A.prototype.constructor` 又是指向誰呢？這個不難判斷，因為實例 ``a.constructor` 默認就是指向 `A.prototype.constructor`，即

```javascript
console.log(a.constructor === A.prototype.constructor);  //true  
console.log(A.prototype.constructor === A);  //true
```

`constructor` 屬性始終指向創建當前物件的構造函數。比如下面例子：

```javascript
var arr = [1, 56, 34, 12];  // 等價於 var foo = new Array(1, 56, 34, 12);  
console.log(arr.constructor === Array); // true  

var Foo = function() { };  // 等價於 var foo = new Function();  
console.log(Foo.constructor === Function); // true  

var obj = new Foo();  // 由構造函數產生實體一個obj物件  
console.log(obj.constructor === Foo); // true 

// 將上面兩段代碼合起來，就得到下面的結論  
console.log(obj.constructor.constructor === Function); // true 
```

但是當 `constructor` 遇到 `prototype` 時，有趣的事情就發生了。
我們知道每個 **函數** 都有一個預設的屬性 `prototype`，而這個 `prototype` 的 `constructor` 預設指向這個函數。如下例所示：

```javascript
function A() {  
    ... 
};  

var a = new A();  

console.log(a.constructor === A);  // true  
console.log(A.prototype.constructor === A); // true  

// 將上兩行代碼合併就得到如下結果  
console.log(a.constructor.prototype.constructor === A); // true 
```

當時當我們重新定義函數的 `prototype` 時（注意：和上例的區別，這裡不是修改而是覆蓋），`constructor` 屬性的行為就有點奇怪了，如下示例：

```javascript
function A() {  
    ...  
};

A.prototype = {  
    fun: function() {  
        ...  
    }  
};  

var a = new A();

console.log(a.constructor === A);  // false  
console.log(A.prototype.constructor === A); // false  
console.log(a.constructor.prototype.constructor === A); // false
```

為什麼呢？原來是因為覆蓋 `A.prototype` 時，等價於進行如下代碼操作：

```javaScript
A.prototype = new Object({  
    fun: function() {  
        ...  
    }  
});
```

而 `constructor` 屬性始終指向創建自身的構造函數，所以此時 `A.prototype.constructor === Object`。
（注意：新覆蓋的物件並不是 **原型物件**，`A.prototype` 本身沒有並 `constructor` 屬性，所以會再往原型鍊上查找 `A.prototype.__proto__.constructor`，找到後回傳 `Object`。）

```javascript
function A() {  
    ... 
};  

A.prototype = {  
    fun: function() {  
        ...
    }  
};  

var a = new A();

console.log(a.constructor === Object);  // true  
console.log(A.prototype.constructor === Object); // true  
console.log(a.constructor.prototype.constructor === Object); // true 
```

怎麼修正這種問題呢？方法也很簡單，重新覆蓋 `A.prototype.constructor` 即可：

```javascript
function A() {  
    ...
};

A.prototype = new Object({  
    fun: function() {  
        ...
    }  
}); 

A.prototype.constructor = A;  
var a = new A();  

console.log(a.constructor === A);  // true  
console.log(A.prototype.constructor === A); // true  
console.log(a.constructor.prototype.constructor === A); // true 
```

## 4. 只有函式有 `prototype` 屬性。

## 5. 所有函式的 `__proto__` 都指向 `Function.prototype` ，它是一個空函數（Empty function）。

```javascript
Number.__proto__ === Function.prototype  // true
Boolean.__proto__ === Function.prototype // true
String.__proto__ === Function.prototype  // true
Object.__proto__ === Function.prototype  // true
Function.__proto__ === Function.prototype // true 
Array.__proto__ === Function.prototype   // true
RegExp.__proto__ === Function.prototype  // true
Error.__proto__ === Function.prototype   // true
Date.__proto__ === Function.prototype    // true
```
`Mat`h、JSON 是以物件形式存在的，無需使用 `new` 方法創建。它們的 `__proto__` 是 `Object.prototype`

```javascript
Math.__proto__ === Object.prototype  // true 
JSON.__proto__ === Object.prototype  // true
```

當然包括自定義的建構函式，

```javascript
function Person() {}
var Man = function() {}
console.log(Person.__proto__ === Function.prototype) // true
console.log(Man.__proto__ === Function.prototype)    // true
```

這說明了，所有的建構函式都來自於 `Function.prototype`，甚至包括根建構物件 `Object` 及` Function` 自身。所有建構函式物件都繼承了 `Function.prototype` 的屬性及方法。如 `length`、`call`、`apply`、`bind`。

## 所有被創建的物件的 `__proto__` 都指向其建構函式的 `prototype` 物件

```javascript
var obj = {name: 'jack'}
var arr = [1,2,3]
var reg = /hello/g
var date = new Date
var err = new Error('exception')

console.log(obj.__proto__ === Object.prototype) // true
console.log(arr.__proto__ === Array.prototype)  // true
console.log(reg.__proto__ === RegExp.prototype) // true
console.log(date.__proto__ === Date.prototype)  // true
console.log(err.__proto__ === Error.prototype)  // true
```

再看看自定義的建構函式

```javascript
function Person(name) {
  this.name = name
}
var p = new Person('jack')
console.log(p.__proto__ === Person.prototype) // true
```

實例 `p` 是用建構函式 `Person` 創建的物件，`p` 的內部原型總是指向其建構函式 `Person` 的 `prototype`。