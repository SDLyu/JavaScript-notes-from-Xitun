# NodeJS 異常處理

## 安全地丟出異常

理想上，我們希望能 **避免** 未被補獲的異常（uncaught errors）發生，與其直接 `throw new Error()` ，藉由下面的幾種方法能安全地丟出異常：

### 同步（Synchronous）異常處理

如果程式碼是同步的，例外異常發生時，直接將錯誤回傳。

#### Return `Error()`

```javascript 
var divideSync = function(x, y){
  if(y === 0){
      return new Error("Can't divide by zero");
  }
  else{
      return x / y;
  }
};

result = divideSync(4, 0);

if(result instanceof Error){
  console.log('4 / 0 = err', result);
}
else{
  console.log('4 / 0 = ' + result);
}
```

### 異步（Asynchronous）異常處理

由於 NodeJS 充斥著大量非同步的回調函數，使得 `try...catch` 無法捕捉所有的例外異常。  

```javascript
try{
  process.nextTick(function(){
    throw new Error();
  });
}
catch (err){
  //can not catch it
}
```

而對於 WEB 服務而言，其實是希望這樣處理例外異常：

```javascript
app.get('/index', function(req, res) {
  try{
    // logic
  }
  catch (err){
    logger.error(err);
    res.statusCode = 500;
    return res.json({success: false, message: 'Server failed'});
  }
});
```

如果 `try…catch` 能捕獲所有的例外異常，這樣就能在出現一些非預期的錯誤時，記錄錯誤，同時友好的給調用者返回一個 500 錯誤。  
可惜，`try…catch` 無法捕獲非同步中的例外異常。所以我們能做的只能是：

```javascript 
app.get('/index', function(req, res){
  // logic
});

process.on('uncaughtException', function(err){
  logger.error(err);
});
```

雖然我們可以記錄下這個例外異常的日誌，且程序也不會異常退出，但是我們沒辦法對例外異常的請求友好地返回，只能夠讓它超時返回。  

+ #### Callback-based

	將 error 傳入回調函數，由回調函數處理。當有例外異常發生時，把它放在回調函數的 `err` 參數（NodeJS 的風格是將 `err` 參數擺在第一個）傳入。   

	```javascript
	var divide = function(x, y, next){
	  if(y === 0){
	    next(new Error("Can't divide by zero"));
	  }
	  else{
	    next(null, x / y);
	  }
	};

	divide(4, 2, function(err, result){
	  if(err){
	    console.log('4 / 2 = err', err);
	  }
	  else{
	    console.log('4 / 2 = ' + result);
	  }
	}); 
	// output 4 / 2 = 2

	divide(4, 0, function(err, result){
	  if(err){
	    console.log('4 / 0 = err', err);
	  }
	  else{
	    console.log('4 / 0 = ' + result);
	  }
	}); 
	// output 4 / 0 = err
	```
	當一切正常時回調函數的 `err` 參數就傳入 `null` ，其它參數則擺在 `err` 後面，這樣一來就可以完美地處理非同步所發生的例外。  
	事實上， NodeJS 的風格也是使用這種方法處理非同步的例外異常。

+ #### EventEmitter trigger

	除了傳遞 `err` 給回調函數的方式之外，還可以用 EventEmitter 觸發特定的 event 來補獲異常事件。

	```javascript
	var events = require('events');
	
	var Divider = function(){
	    events.EventEmitter.call(this);
	};  

	require('util').inherits(Divider, events.EventEmitter);

	Divider.prototype.divide = function(x, y){
	    if(y === 0){
	        var err = new Error("Can't divide by zero");
	        this.emit('error', err);
	    }
	    else{
	        this.emit('divided', x, y, x / y);
	    }

	    return this;
	};

	var divider = new Divider();

	divider.on('error', function(err){
	    console.log(err);
	});

	divider.on('divided', function(x, y, result){
	    console.log(x + '/' + y + '=' + result);
	});

	divider.divide(4, 2).divide(4, 0);
	```
	實例化 `Divider` 後，必須透過 `on` 綁定 handler function 。  
	當有例外異常發生時 `emit` 一個 `error` 事件，接著把 `err` 做為參數傳入 event handler；當一切正常時 `emit` 一個 `divided` 事件，同時把 `result` 傳入 event handler。

## 安全地補獲異常

有時候，如果我們沒有將錯誤補獲，可能會產生 `uncaughtException` 甚至使整個應用程序當掉。  
透過下面幾種方法，能將例外異常補獲住：

+ ### `try...catch`

	如果這個例外異常是發生在 **同步** 的程式碼中，最簡單的方法是使用 `try...catch` 包覆就能補獲例外異常並處理。  

	```javascript
	try{
	  var err = new Error('Catch error!');
	  throw err;
	}
	catch(err){
	  console.log(err);
	}
	```

+ ### Domain module

	在 NodeJS v0.8+ 版本的時候，發佈了一個模組 domain。這個模組做的就是 `try…catch` 無法做到的： **捕捉非同步回調中出現的異常** 。  
	於是乎，我們上面那個無奈的例子就有了解決的方案：

	```javascript
	var domain = require('domain');

	//加入一個 domain 的 middleware，將每個請求都包裹在一個獨立的 domain 中來處理非同步的異常
	app.use(function(req, res, next){
	  var d = domain.create();
	  //監聽 domain 的錯誤事件
	  d.on('error', function(err){
	    logger.error(err);
	    res.statusCode = 500;
	    res.json({sucess:false, messag: 'Server failed'});
	    d.dispose();
	  });

	  d.add(req);
	  d.add(res);
	  d.run(next);
	});

	app.get('/index', function(req, res){
	  // logic
	});
	```

	通過中間件的形式，引入 `domain` 來處理非同步中的異常。`domain` 雖然捕捉到了異常，如果這個異常會導致堆疊丟失，進而引起記憶體洩漏時，這種情況還是需要重啟程序的，有興趣可以去看看 domain-middleware 這個 `domain` 中間件。

	+ #### `domain` 解析

		回過頭來，我們來看看 `domain` 做了些什麼來讓我們捕獲非同步的請求（代碼來自 NodeJS v0.10.4 ，此部分可能正在快速變更）。如果對 `domain` 還不甚瞭解的，可以先簡單看過 `domain` 的文檔。

	+ #### NodeJS 事件循環機制

		```javascript
		function laterCall(){
		  console.log('print me later');
		}

		process.nextTick(laterCallback);
		console.log('print me first');
		```

	+ #### `domain` 的實現

		在瞭解了 NodeJS 的事件堆疊機制之後，我們再來看看 `domain` 做了些什麼事。  
		`domain` 其實是一個 `EventEmitter` 物件，它通過事件的方式傳遞捕獲的錯誤。這樣我們在研究它的時候，就簡化到兩個點：

		+ ##### 什麼時候觸發 `domain` 的 `error` 事件：

			如果程序拋出了異常，但沒被任何的 `try…catch` 捕獲到，這時候將會觸發整個`process` 的`processFatal` ，如果程序被包裹在 `domain`之中，則會在 `domain` 上觸發 `error` 事件。  
			相反地，若沒被包裏在 `domain` 之中，將會在 `process` 上觸發 `uncaughtException` 事件。  

		+ ##### `domain` 如何在多個不同的事件堆疊中傳遞：  

			+ 當 `domain` 被實例化之後，我們通常會調用它的 `run` 方法（如之前在 WEB 服務中的使用），將某個函數包裏在這個 `domain` 實體中並執行。被包裹的函數在執行的時候， `process.domain` 這個全域變數將會被指向這個 `domain` 實體。當這個事件堆疊中，拋出異常調用 `processFatal` 的時候，發現 `process.domain` 存在，就會在 `domain`n 實體上觸發 `error` 事件。  

			+ 在 `require` 引入 `domain` 模組之後，會重寫全域的 `nextTick` 和 `_tickCallback` ，注入一些 `domain` 相關的代碼：

				```javascript
				//簡化後的domain傳遞部分代碼
				function nextDomainTick(callback){
				  nextTickQueue.push({callback: callback, domain: process.domain});
				}

				function _tickDomainCallback(){
				  var tock = nextTickQueue.pop();
				  //設置process.domain = tock.domain
				  tock.domain && tock.domain.enter();
				  callback();
				  //清除process.domain
				  tock.domain && tock.domain.exit();        
				}
				```

				這是在多個事件迴圈中傳遞 `domain` 的關鍵： 使用`nextTick` 加入事件佇列的時候，會記錄當前的 `domain` 。當事件佇列被 `_tickCallback` 調用的時候，將新的事件迴圈的 `process.domain` 設置為之前記錄的 `domain` 。  
			如此一來，在被 `domain` 所包裹的程式碼中，不管如何調用 `process.nextTick `， `domain` 將會一直被傳遞下去。  

			+ 當然， NodeJS 的非同步還有兩種情況，一種是 `event` 形式。因此在 `EventEmitter`的構造函數有如下代碼：  

				```javascript
				if (exports.usingDomains{
				  // if there is an active domain, then attach to it.
				  domain = domain || require('domain');
				  if (domain.active && !(this instanceof domain.Domain){
				    this.domain = domain.active;
				  }
				}
				```

				實例化 `EventEmitter` 的時候，將會把這個物件和當前的 `domain` 綁定，當通過 `emit` 觸發這個物件上的事件時，像 `_tickCallback` 執行的時候一樣，回調函數將會重新被當前的 `domain` 包裹住了。

			+ 而另一種情況，是 `setTimeout` 和 `setInterval` ，同樣的，在 timer 的源始碼中，我們也可以發現這樣的一句程式碼:

				```javascript
				if(process.domain) timer.domain = process.domain;
				```

				跟 `EventEmmiter` 一樣，之後這些 timer 的回調函數也將被當前的 `domain` 包裹住了。

+ ### Catch `uncaughtException`

	最後一種方式就是捕獲 `uncaughtException` 在 process 模組上可以綁定一個 `uncaughtException` 事件

	```javascript
	process.on('uncaughtException', function(err) {
    	console.log(err);
	});

	throw new Error('Catch error!'); // output Catch error!
	```

	不過不建議綁定 `uncaughtException` 來處理例外異常，因為官方文件有說明：

	> Note that uncaughtException is a very crude mechanism for exception handling and may be removed in the future.  
	> 請住意到 uncaughException 是一種非常粗魯的例外異常處理機制，在將來也許會被移除。