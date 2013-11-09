# NodeJS 心得

## 1. module 模組的實現

NodeJS 中 module 模組的實現其實是依賴於閉包的，也就是說， `module` ， `exports` 其實都是外部傳入的參數。 
整個模組可以簡化成：　

```javascript
function NativeModule(id){
    this.id=id;
    this.filename=id+".js";
    this.exports={};
}

NativeModule.require=function(id){
    var module=new NativeModule(id);
    module.compile();
    return  module.exports();
};

NativeModule.prototype.compile = function() {  
   (function(exports, require, module, __filename, __dirname){
        exports=module.exports=function(){alert("leexiaosi")};
   }(this.exports, NativeModule.require, this, this.filename));
};

NativeModule.require("leexiaosi")
```

通過上面的程式碼可以發現，其實我們看見的模組文件的代碼，一般而言都是類似

```javascript
exports = module.exports = function(){ alert("leexiaosi") };
```

同時也就明白，為什麼在模組文件中用 `var` 定義的變量都是私有的，其關鍵點是這個函數的實現。

```javascript
NativeModule.prototype.compile
```

為了方便理解，我們設

```javascript
var fnImport = function(exports, require, module, __filename, __dirname)
```

根據閉包的原理，這個函數在執行時，其 `[[scpoe]]` 會記錄 `fnImport` 這個函數定義的作用域的各個變量。

+ `fnImport` 函數的形參（即 `this.exports`，`NativeModule.require`，`this`，`this.filename` ）。
+ `fnImport` 函數中`function`宣告的函數。
+ `fnImport` 函數中通過`var`宣告的變量。
+ `fnImport` 函數的父函數的作用域`[[scope]]`。

`fnImport` 這個函數對其 `fnImport.[[scope]]` 的操作都是有權限的。故 `this.exports` 會被更改，儘管在 `NativeModule` 的建構式中 `this.exports` 的定義是空物件。

從 `return module.exports()` 來看， `require` 初次加載模組時，必然是阻塞的(初次加載後會被緩存，所以加載之後再 `require` 就不再是阻塞的了)。

## 2. `exports` 與 `module.exports` 的差別

```javascript 
//fun1.js
module.exports = function(){...}

//fun2.js
exports.fun2 = function(){...}

//fun3.js
exports = function(){...}

//app.js
var fun1 = require('./fun1');
var fun2 = require('./fun2');
var fun3 = require('./fun3');

fun1();      // will work
fun2.fun2(); // will work
fun3();      // will fail

```

根據 Node.js 裡 module 模組實現，

```javascript 
  exports = module.exports = function(){...};
```

+ `exports` 和 `module.exports` 都引用到同一個對象。
+ 呼叫 `require()` 時所引用的是 `module.exports` 對象。

1. 當修改了 `module.exports` 的引用時， `exports` 的引用並沒有改變，所以 `moduel.exports = function(){...}` 是可行的。  
2. 覆寫 `exports = function(){...}` 時，改變了 `exports` 的引用， `require()` 始終引用 `module.exports = {}` ，所以 `require` 回傳 `{}` 空物件。
3. 比較好的編程習慣是指定 `exports = module.exports` ，直接覆蓋 `module.exports` 不是一個好習慣。

## 3. 理解 Express 的 `app.get()`

Express 裡有兩個 `app.get()` 的方法，分別帶有一個型參及多個型參的函數。

**app.get(name)**
> Get setting name value.

```javascript
app.get('title');
// => undefined

app.set('title', 'My Site');
app.get('title');
// => "My Site"
```

**app.VERB(path, [callback...], callback)**
> The app.VERB() methods provide the routing functionality in Express, where VERB is one of the HTTP verbs, such as app.post().

從 Express 的 [application.js] 原始碼實現來看

```javascript
methods.forEach(function(method){
  app[method] = function(path){
    if ('get' == method && 1 == arguments.length) return this.set(path);
    var args = [method].concat([].slice.call(arguments));
    if (!this._usedRouter) this.use(this.router);
    return this._router.route.apply(this._router, args);
  }
});
```

若調用 `app.get()` 時只有一個形參，則判斷是取設定值，否則判斷註冊路由。

```javascript
if ('get' == method && 1 == arguments.length) return this.set(path); 
```
也許你會有疑問，實現取值不應該是 `return this.get(path)` 嗎，怎麼會是用 `set()` ？  
如果改成 `get(path)` 就會變成無窮遞迴，所以在 `set()` 的實現上，若調用 `set()` 時只有一個形參則判斷是取值。

## 4. 什麼是 Node.js 的 Connect 、 Express 和 middleware ？

原文來自 StackOverflow 的文章 [What is Node.js' Connect, Express and “middleware”?][wm]，以下是翻譯。  
我很高興有人問這個問題，因為對於 Node.js 開發者而言，肯定都會有這樣的疑問。以下是我的一些見解：  

+ Node.js 本身提供了一個 [http] 模組，它的 `createServer` 方法回傳一個繼承自 `http.Server` 原型的物件，讓你可以用來回應 HTTP 請求。
+ [Connect] 也提供了一個 `createServer` 方法，它回傳一個繼承 `http.Server` 擴張版的物件。 Connect 的擴張使自已能容易插入（plug in）一些中間件（[middleware]）。這也是為什麼 Connect 將自已視為一個 "middleware framework" 。類推到 Ruby ，就像是 Ruby 的 Rack。
+ [Express] 與 Connect 的關係就像 Connect 與 http 模組的關係： Express 提供一個擴張 Connect 的 `Server`原型的`createSerer`方法。所以 **所有的 Connect 功能都包在 Express 裡面** ，且又增加了視圖描繪（view rendering）和便利的DSL路由。可以類推到 Ruby 的 Sinatra 。 
+ 也有一些擴展 Express 的框架， [Zappa] 整合了對 CoffeeScript 的支援，伺服器端的 jQuery 和測試功能。

下面有實際的範例告訴你什麼是中間件：另一個角度來看，上面的框架都不能幫你處理靜態的檔案。但如果將他丟入　`connect.static`（一個 Connect 的中間件），將設定好目錄，你的伺服器就能存取目錄裡的檔案。 EXPRESS 也提供 Connect 的中間件； `express.static` 與 `connect.static` 提供一樣的功能。

最近，大多數的 Node.js 應用程式都是使用 Express 開發而成的；也說明它所增加的功能很實用，而且所有底層的功能也都保留著。

## 5. `app.use()` 與中間件

`app.use([path], function)`
> Use the given middleware `function`, with optional mount path, defaulting to "/". <br>
> 使用 `function` 中間件，和一個可選的路徑參數，預設為 "/"。

Connect 裡面有一個十分好用的應用概念稱為中間件，可以透過中間件做出複雜的效果。

```javascript
// .. create http server

app.use(express.bodyParser());           //use bodyParser middleware
app.use(express.methodOverride());       //use methodOverride middleware
app.use(express.session());              //use session middleware

app.use(function(req, res, next){...});  //use self-defined middleware
```

上面都是中間件的使用方式，透過 `app.use()` 方法載入中間件函式執行。處理函式包含三個基本參數， response 、 request 和 next，其中 next 表示下一個中間件 函式，同時會自動將預設三個參數繼續帶往下個中間件函式執行。所有 url 皆會依 `app.use() `設定的順序執行中間件。

Connect 提供的中間件（Express 保留 Connect 的中間件）：

+ `logger()`：輸出日志。
+ `bodyParser()` ：可以對客戶端所提交的 POST 請求進行解析並放入 request.body 中。默認支持：`application/json`，`application/x-www-form-urlencoded` 以及 `multipart/form-data`，但不支持 XML 型式的請求。默認情況下， Express 並不知道該如何處理請求的body，因此需要加入 `bodyParser()` 用於分析請求的body。
+ `basicAuth()`：提供基礎的HTTP驗證。
+ `methodOverride()` ：可以協助處理 POST 請求偽裝成 PUT、DLETE 或其它 HTTP 方法（因為 XHTML 1.x 的表單只有支持 POST 和 GET）。
+ `express.static(__dirname +'/public')` ：用來處理靜態的請求，例如 CSS、js、img等，路徑中（`__dirname+'/public'`）的靜態文件會被直接作為靜態資源輸出。
+ `errorHandler()` 是 Conncet 內建的中間件協助處理例外，並可帶入一些參數　`{ dumpExceptions:true, showStack:true}`。 dumpExceptions 能在開發環境輸出異常到 `stderr`， showStack 可以在 HTML 頁面顯示異常。

更多中間件請參考[Connect]。

## 6. 使用 PUT 、 DELETE 等 HTTP 方法

Connect 有一個中間件（`methodOverride`）能提供表單偽裝成 PUT 或其它方法。

客戶端：
```html
<form method ='POST' action='/user/id'> ...
  <input type="hidden" name="_method" value="put" />
</form>
```

後端：
```javascript
app.put('/users/:id', function (req, res, next) {
  // edit your user here
});
```



[application.js]: https://github.com/visionmedia/express/blob/master/lib/application.js
[wm]: http://stackoverflow.com/questions/5284340/what-is-node-js-connect-express-and-middleware
[http]: http://nodejs.org/docs/v0.4.2/api/all.html#hTTP
[Connect]: http://www.senchalabs.org/connect/
[middleware]: https://github.com/senchalabs/connect/wiki
[Zappa]: https://github.com/mauricemach/zappa
[Express]: http://expressjs.com/
[Connect]: http://www.senchalabs.org/connect/