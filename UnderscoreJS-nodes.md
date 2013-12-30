# UnderscoreJS 心得

## UnderscoreJS 的程式碼分析

UnderscoreJS 先將所有的函數宣告在前面  

```javascript
_.map = _.collect = function(obj, iterator, context) {
	...

_.reduce = _.foldl = _.inject = function(obj, iterator, memo, context) {
	...
```

接著在最下面，將所有的函數加到 `_` 的原型上並綁定 `_wrapper`

```javascript
_.mixin = function(obj) {
	each(_.functions(obj), function(name) {
		var func = _[name] = obj[name];
		_.prototype[name] = function() {
			var args = [this._wrapped];
			push.apply(args, arguments);
			return result.call(this, func.apply(_, args));
		};
	});
};

	...

// Add all of the Underscore functions to the wrapper object.
_.mixin(_);

// Add all mutator Array functions to the wrapper.
each(['pop', 'push', 'reverse', 'shift', 'sort', 'splice', 'unshift'], function(name) {
	var method = ArrayProto[name];
	_.prototype[name] = function() {
		var obj = this._wrapped;
		method.apply(obj, arguments);
		if ((name == 'shift' || name == 'splice') && obj.length === 0) delete obj[0];
		return result.call(this, obj);
	};
});

// Add all accessor Array functions to the wrapper.
each(['concat', 'join', 'slice'], function(name) {
	var method = ArrayProto[name];
	_.prototype[name] = function() {
		return result.call(this, method.apply(this._wrapped, arguments));
	};
});
```

