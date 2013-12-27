# Javascript 良好程式收集

## 1. 只搜尋父類別的屬性

使用 `in` 搜尋屬性時，會沿著 prototype chain 搜尋。可以加上 `hasOwnProperty` 方法過濾祖父類別的屬性。

```javascript
var child = new parent();

for(var prop in child){
    if(parent.hasOwnProperty(child[prop]))
        console.log(prop);
}
```

## 2. 將判斷與陳述式結合

JavaScript 的 and 和 or 只要符合條件就不繼續檢查。因此在陳述式前加上或（`||`）且（`&&`），能更簡潔地將判斷與陳述式結合。
假設自己寫的 JS 框架使用某個符號（變數）作為簡寫（如 `$`），但又不想跟其他框架衝突（如 jQuery）時，可以先判斷該符號（變數）是否已存在，存在則沿用舊的，不存在就初始符號（變數）。

```javascript
(window.$) ? window.$ : jQuery;
```
上面這種寫法也行但更常看到的是下面這種寫法。

```javascript
window.$ = window.$ || jQuery;
```

## 3. 永遠使用模組建構程式

JavaScript 中沒有真正的模組法，但是我們可以將程式碼包在匿名立即函式（JavaScript 以 `function(){}` 的作為區塊）裡來達到模組的效果。  
常見的 JavaScript 的框架都會使用到模組，避免對全域變數造成影響。

```javascript
//impress.js
(function(){
    var impress = window.impress = function ( rootId ){
        ...
    };
})();
```

## 4. 將全域物件傳入模組

JavaScript 查找變數的規則是，會先從區域變數找起，再往上找，最終查找到全域物件。  
將全域變數傳入模組有以下好處：

+ __加快變數查找速度__：因為傳入的全域物件參數，已經變成了區域變數了，不會再往上查找。
+ __匯出全域變數__：區域變數 `window` 仍然指向全域物件，因此在 `window` 變數上新增的屬性依然是全域變數。以後在模組內也能產生全域變數了。

```javascript
//impress.js
(function(document, window){
    var impress = window.impress = function ( rootId ){
        ...
    };
})(document, window);
```

## 5. `Underscore` 的安全引用物件（保證物件只會被包裏一次）

```javascript  
var _ = function(obj) {
    if (obj instanceof _) return obj;
    if (!(this instanceof _)) return new _(obj);
    this._wrapped = obj;
};
```

Line1：`_(obj)` 若 `obj` 已使用 `-` 呼叫初始過則直接回傳 `obj`。  
Line2：若 `this` 不是 `_` 的實例，則呼叫 `_` 包裏 `obj` 並回傳。  
Line3：當呼叫 `new _(obj)` 時，設定 `_wrapped` 的屬性為 `obj`。  

Line2 的條件式不成立時，會重新綁定 `_wrapped`。

```javascript
var unobj = _(obj);
unobj._();      // 重新綁定 _wrapped 至 unobj
unobj._(unobj); // 重新綁定 _wrapped 至 unobj
```
