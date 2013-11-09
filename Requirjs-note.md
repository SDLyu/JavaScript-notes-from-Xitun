# RequireJS

> RequireJS is a JavaScript file and module loader. It is optimized for in-browser use, but it can be used in other JavaScript environments, like Rhino and Node. Using a modular script loader like RequireJS will improve the speed and quality of your code.  
RequireJS 是一個檔案和模組的載入器。最適在瀏覽器端使用，但也能運行於其它的 JavaScript 環境，像 Rhino 和 Node。使用一個模組腳本載入器能提升程式碼的速度及品質。  

ReauireJS 採用了和傳統的通過 script 標籤載入腳本檔案不同的方法，目標就是實現模組模式。模組模式可以帶來更好的性能和可維護性。讓我們來看一下 RquireJS 是如何使用的。

## 加載模組

RequireJS 以一個相對於 baseUrl 的地址來載入所有的代碼。 頂層的 `<script>` 標籤含有一個特殊的屬性 data-main ， require.js 使用它做為腳本的載入點，而 baseUrl 一般設置成與該屬性一致的目錄。  

一開始先在 html 中引入 require.js 並設置 data-main 屬性

```html
<!-- 將 baseUrl 設置到 "scripts" 目錄，並且載入 module ID 為 'main' 的一個腳本-->
<script data-main="scripts/main.js" src="scripts/require.js"></script>
```

當 HTML 檔案中的 `data-main` 和 `src` 都設置好後，來看一下在 main.js 檔案中如何加載模組。  
載入模組使用 `require` 函數  

```javascript
require('ModuleName', dependencies, callback);
```

### 參數說明：

`ModuleName`：一個可選的模組名稱。  
`dependencies`： 依賴的模組路徑用陣列 `[]` 方式引入。  
`callback`：當模組完成載入後，執行回呼函數。

### 範例
```javascript
require(['lib/a'], function(moduleA){
    moduleA.doSomething();
});
```

接著來看看 RequireJS 如何定義模組  

## 定義模組

模組不同於傳統的指令檔，它定義了一個作用域來避免全域名稱空間污染。它可以顯式地列出其依賴關係，並以函數（定義此模組的那個函數）參數的形式將這些依賴進行注入，而無需引用全域變數。

RequireJS 的模組語法允許它儘快地載入多個模組，雖然載入的順序不定，但依賴的順序最終是正確的。  
RequireJS 實作 AMD 所定義的 `define` API 方法，所以我們就可以用它來實現程式的模組化。  
`define` 的函數定義如下：

```javascript
define(id?, dependencies?, factory);
```

### 參數說明：

`id`：可選的模組名稱，相對於目前的檔案路徑。  
`dependencies`：依賴的模組路徑用陣列 `[]` 方式引入。  
`factory`：一個工廠方法，它必須回傳一個物件或函數。  

### 範例：

#### 定義簡單的物件

如果一個模組僅含值對，沒有任何依賴，則在 `define()` 中直接定義這些值對即可：

```javascript
define({
    color: "black",
    size: "unisize"
});
```

#### 定義函數

如果一個模組沒有任何依賴，但需要一些前置的操作。則將操作定義在函數中，並將其傳給 `define()`：

```javascript
define(function () {
    //Do setup work here

    return {
        color: "black",
        size: "unisize"
    }
});
```

#### 定義存在依賴的函數

如果模組存在依賴：則第一個參數是依賴的模組陣列；第二個參數是函數，在所有依賴載入完畢後，該函數會被調用，並返回一個物件。依賴關係會以參數的形式傳進函數裡，參數清單與依賴名稱列表相互對應。

```javascript
define(["./cart", "./inventory"], function(cart, inventory) {
        return {
            color: "blue",
            size: "large",
            addToCart: function() {
                inventory.decrement(this);
                cart.add(this);
            }
        }
    }
);
```

#### 將模組定義成一個函數

(未完成)

模組的返回值型別並沒有強制是物件，任何函數的返回值都是允許的。

```javascript
define(["my/cart", "my/inventory"],
    function(cart, inventory) {
        return function(title) {
            return title ? (window.title = title) :
                   inventory.storeName + ' ' + cart.name;
        }
    }
);
```

#### 包裝成 CommonJS 來定義模組

如果你有一些以 CommonJS 模組模式編寫的程式碼（node的模組即是如此），這會使得這些模組難以使用上述的方法定義，而你也不想重構這些程式碼。  
RequireJS 提供了一個 CommonJS 的簡單包裝

```javascript
define(function(require, exports, module) {
        var a = require('a'),
            b = require('b');

        //Return the module value
        return function () {};
    }
);
```

#### 定義一個命名模組

## RequireJS 與第三方套件

我們直接利用該套件定義好的 namespace ，例如 jQuery 的 `$` 符號，或是 underscore.js 的 `_` 符號。

```javascript
require(['lib/jquery', 'libs/underscore', 'libs/backbone'], function(){
    console.log($); // work
    console.log(_); // work
    console.log(Backbone); //work
});
```

我們並沒有再為這些套件指定新的 namespace ，是因為這些 namespace 已經被綁在 `global` 變數裡了。   
再看一個例子，若我們將套件的別名傳進函數裡

```javascript
require(['lib/jquery', 'libs/underscore', 'libs/backbone'], function($, _, Backbone){
    console.log($); // work
    console.log(_); // work
    console.log(Backbone); //undefined
});
```
會發現傳進函數裡的參數都變成 `undefined`，所以很多人在透過 RequireJS 使用這些套件時，就會在這裡卡關。  
為什麼呢？主要是因為這 jQuery、Underscore 這兩個套件已支援 AMD 架構。

```javascript
// Expose jQuery as an AMD module, but only for AMD loaders that
// understand the issues with loading multiple versions of jQuery
// in a page that all might call define(). The loader will indicate
// they have special allowances for multiple jQuery versions by
// specifying define.amd.jQuery = true. Register as a named module,
// since jQuery can be concatenated with other files that may use define,
// but not use a proper concatenation script that understands anonymous
// AMD modules. A named AMD is safest and most robust way to register.
// Lowercase jquery is used because AMD module names are derived from
// file names, and jQuery is normally delivered in a lowercase file name.
// Do this after creating the global so that if an AMD module wants to call
// noConflict to hide this version of jQuery, it will work.
if ( typeof define === "function" && define.amd && define.amd.jQuery ) {
    define( "jquery", [], function () { return jQuery; } );
}
```
當 `define` 這個函式已經定義時， jQuery 把自已輸出成一個命名模組。  
Underscore 也做了類似的事，但是Backbone 並不支援 AMD 架構。

### 坑爹的地方

照理說，只要支援了 AMD 架構的套件，應該就能像上例一樣，將別名傳入參數就行了

```javascript
require(['lib/jquery', 'libs/underscore', 'libs/backbone'], function($, _){
    console.log($); // undefined
    console.log(_); // undefined
});
```

但卻還是 `undefined`！！？  
原因是 jQuery 和 Underscore 預設的載入位置是與當前檔案同目錄

```javascript
define( "jquery", [], function () { return jQuery; } );
```

### 解決辦法

+ 將 main.js 和 jquery.js 放在同一個目錄底下。
+ 修改 jquery.js 的程式碼，將 `define` 裡的名稱修改成 jquery.js 的目錄。

    ```javascript
    if ( typeof define === "function" && define.amd && define.amd.jQuery ) {
        define( "libs/jquery", [], function () { return jQuery; } );
    }
    ```
+ 設置 `require.config()` 的 `baseUrl`屬性。

    ```javascript
    require.config({
        baseUrl: '/lib',
    });
    ```

## 順序問題

因為使用非同步的載入方式，所以用 `require` 載入套件時，是有可能會造成相依性上的問題。 所幸 RequireJS 提供了一個 `order plugin` ，讓我們可以依序載入正確的套件。
(未完成)

## RequireJS 風格

+ 一個檔案只包含一個模組，也就是說只定義一次 `define()` 。
+ 以 translating CommonJS modules 的形式以一个更簡短的名字變量來引用模組

    ```javascript
    define(function(require) {
        var mod = require("./relative/name");
    });
    ```
+ 你可以自定義模組名稱（jquery 函式庫就有自定義模組名稱），但這使得模組更不具備移植性——若你將模組檔案移動到另一個資料夾下，你就得重新命名。