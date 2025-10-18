
# Proxy ve Reflect

Bir *proxy* (vekil), başka bir nesneyi sarmalar ve özellik okuma/yazma gibi işlemleri engeller (intercept eder). Bu işlemleri kendisi yönetebilir veya şeffaf bir şekilde hedef nesnenin yönetmesine izin verebilir.

Proxy’ler birçok kütüphanede ve bazı tarayıcı framework’lerinde kullanılır. Bu bölümde birçok pratik kullanım örneği göreceğiz.

Sözdizimi:

```js
let proxy = new Proxy(target, handler)
```

- `target` -- sarmalanacak nesne; fonksiyonlar da dahil olmak üzere herhangi bir şey olabilir.
- `handler` -- işlemleri yakalayan (“trap” adı verilen) metotları içeren bir nesnedir. Örneğin, bir özelliği okumak için `get`, bir özelliğe yazmak için `set` gibi.

`proxy` üzerinde bir işlem yapılırsa, `handler` içinde o işleme karşılık gelen bir trap varsa, o çalıştırılır; yoksa işlem `target`üzerinde gerçekleştirilir.

Basit bir örnek olarak, hiç trap içermeyen bir proxy oluşturalım:

```js run
let target = {};
let proxy = new Proxy(target, {}); // boş handler

proxy.test = 5; // proxy’ye yazma (1)
alert(target.test); // 5, özellik target üzerinde belirdi!

alert(proxy.test); // 5, proxy’den de okuyabiliyoruz (2)

for(let key in proxy) alert(key); // test, döngü çalışıyor (3)
```

Hiç trap olmadığından, `proxy` üzerindeki tüm işlemler `target`’a yönlendirilir.

1. `proxy.test=` yazma işlemi, değeri `target` üzerine yazar.
2. `proxy.test` okuma işlemi, değeri `target`’tan döndürür.
3. `proxy` üzerinde döngü yapmak, `target`’taki değerleri döndürür.

Gördüğümüz gibi, trap olmadan `proxy`, `target` üzerinde şeffaf bir sarmalayıcı gibi davranır.

![](proxy.svg)  

Proxy, özel bir “egzotik nesnedir”. Kendi özellikleri yoktur. Boş bir handler ile, işlemleri tamamen `target`’a yönlendirir.

Eğer sihirli bir davranış istiyorsak, trap’ler eklememiz gerekir.

[Proxy spesifikasyonu](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots)’nda tanımlanmış bir dizi dahili nesne işlemi vardır. Bir proxy, bunlardan herhangi birini yakalayabilir; bunun için handler’a karşılık gelen metodu eklememiz yeterlidir.

Aşağıdaki tabloda:
- **İçsel Metot(Internal Method)** Spesifikasyondaki dahili işlemin adıdır. Örneğin, `[[Get]]` bir özelliği okuma işlemidir.
- **Handler Metodu(Handler Method)** `handler`’a eklememiz gereken metot adıdır; bu metot işlemi yakalar ve özel bir davranış tanımlar.


| İçsel Metot | Handler Metodu | Yakalanan İşlem(Traps)... |
|-----------------|----------------|-------------|
| `[[Get]]` | `get` | özelliği okuma |
| `[[Set]]` | `set` | özelliğe yazma |
| `[[HasProperty]]` | `has` | `in` operatörü |
| `[[Delete]]` | `deleteProperty` | `delete` operatörü |
| `[[Call]]` | `apply` | fonksiyon çağrısı |
| `[[Construct]]` | `construct` | `new` operatörü |
| `[[GetPrototypeOf]]` | `getPrototypeOf` | [Object.getPrototypeOf](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getPrototypeOf) |
| `[[SetPrototypeOf]]` | `setPrototypeOf` | [Object.setPrototypeOf](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf) |
| `[[IsExtensible]]` | `isExtensible` | [Object.isExtensible](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/isExtensible) |
| `[[PreventExtensions]]` | `preventExtensions` | [Object.preventExtensions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/preventExtensions) |
| `[[GetOwnProperty]]` | `getOwnPropertyDescriptor` | [Object.getOwnPropertyDescriptor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyDescriptor) |
| `[[DefineOwnProperty]]` | `defineProperty` | [Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty), [Object.defineProperties](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperties) |
| `[[OwnPropertyKeys]]` | `ownKeys` | [Object.keys](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys), [Object.getOwnPropertyNames](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyNames), [Object.getOwnPropertySymbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertySymbols), yineleme anahtarları |

```warn header="Invariants"
JavaScript, bazı değişmez kuralları zorunlu kılar — bu kurallar, içsel metotlar ve trap’lerin belirli koşulları yerine getirmesini sağlar.

Çoğu, dönüş değerleriyle ilgilidir:
- `[[Set]]` başarılıysa `true`, değilse `false` döndürmelidir.
- `[[Delete]]` başarılıysa `true`, değilse `false` döndürmelidir.
- …ve benzerleri; aşağıda örneklerde göreceğiz.

Bazı diğer kurallar:
- `[[GetPrototypeOf]]` çağrıldığında, proxy nesnesinin prototipi, hedef nesneninkiyle aynı olmalıdır.

Yani, bir `proxy`’nin prototipini okuduğumuzda, her zaman hedef nesnenin prototipini döndürmelidir. `getPrototypeOf` trap bu işlemi yakalayabilir ama bu kurala uymalıdır.

Bu değişmez kurallar, dilin tutarlı ve doğru çalışmasını sağlar. Tüm liste [spesifikasyonda](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots) bulunur; genellikle sıradışı bir şey yapmadığınız sürece ihlal etmezsiniz.

```

Haydi bunun pratik örneklerde nasıl çalıştığına bakalım.

## "get" tuzağı ile varsayılan değer

En yaygın trap’ler özellik okuma/yazma içindir.

Okumayı yakalamak için, `handler` içinde `get(target, property, receiver)` adlı bir metot bulunmalıdır.

Bir özellik okunduğunda tetiklenir:

- `target` -- hedef nesnedir; `new Proxy`’ye ilk argüman olarak verilen nesne,
- `property` -- özellik adı,
- `receiver` -- eğer özellik bir getter ise, o kodda `this` olarak kullanılacak nesnedir. Genellikle bu `proxy` nesnesinin kendisidir (veya proxy’den miras alıyorsak ondan türeyen nesne).

Bir nesne için varsayılan değerleri uygulamak üzere `get`’i kullanalım.

Örneğin, sayısal bir dizinin var olmayan indeksler için `undefined` yerine `0` döndürmesini istiyoruz.

Bunu, okumayı yakalayan ve öyle bir özellik yoksa varsayılan değer döndüren bir proxy ile saralım:


```js run
let numbers = [0, 1, 2];

numbers = new Proxy(numbers, {
  get(target, prop) {
    if (prop in target) {
      return target[prop];
    } else {
      return 0; // varsayılan değer
    }
  }
});

*!*
alert( numbers[1] ); // 1
alert( numbers[123] ); // 0 (böyle bir değer yok)
*/!*
```

Bu yaklaşım geneldir. "Varsayılan" değer mantığını `Proxy` ile istediğimiz gibi kurabiliriz.

Diyelim ki elimizde ifadeler ve onların çevirilerinden oluşan bir sözlük var:

```js run
let dictionary = {
  'Hello': 'Hola',
  'Bye': 'Adiós'
};

alert( dictionary['Hello'] ); // Hola
alert( dictionary['Welcome'] ); // undefined
```

Şu anda, bir ifade yoksa `dictionary`’den okumak `undefined` döndürüyor. Ama pratikte, çevrilmemiş bir ifadeyi olduğu gibi bırakmak çoğu zaman `undefined`’dan daha iyidir. O hâlde, `undefined` yerine varsayılan değerin çevrilmemiş ifadenin kendisi olmasını sağlayalım.

Bunu başarmak için, okumayı yakalayan bir proxy ile`dictionary`’yi saracağız:

```js run
let dictionary = {
  'Hello': 'Hola',
  'Bye': 'Adiós'
};

dictionary = new Proxy(dictionary, {
*!*
  get(target, phrase) { // dictionary’den bir özellik okunmasını yakala
*/!*
    if (phrase in target) { // sözlükte varsa
      return target[phrase]; // çeviriyi döndür
    } else {
      // yoksa, çevrilmemiş ifadeyi döndür
      return phrase;
    }
  }
});

// Sözlükte rastgele ifadeleri ara!
// En kötü ihtimalle çevrilmemiş olarak dönerler.
alert( dictionary['Hello'] ); // Hola
*!*
alert( dictionary['Welcome to Proxy']); // Welcome to Proxy (çeviri yok)
*/!*
```

````smart header="Proxy, her yerde `target` yerine kullanılmalıdır"
Proxy’nin değişkenin üzerine nasıl yazdığına dikkat edin:

```js
dictionary = new Proxy(dictionary, ...);
numbers = new Proxy(numbers, ...);
```

Proxy, hedef nesnenin yerini her yerde tamamen almalıdır. Bir nesne proxylenmişse, sonrasında kimse hedef nesneye doğrudan referans vermemelidir. Aksi takdirde işler kolayca karışır.
````

## "set" tuzağı ile doğrulama (Validation)

Şimdi yazma işlemlerini de yakalayalım.

Diyelim ki yalnızca sayılardan oluşan bir dizi istiyoruz. Eğer farklı türde bir değer eklenirse, bir hata fırlatılmalı.

`set` tuzağı (trap), bir özellik yazıldığında tetiklenir: `set(target, property, value, receiver)`

- `target` -- hedef nesnedir; `new Proxy`’ye ilk argüman olarak verilen nesne,
- `property` -- özellik adı,
- `value` -- özellik değeri,
- `receiver` -- `get` tuzağındakiyle aynıdır; yalnızca özellik bir setter ise önemlidir.

`set` tuzağı, işlem başarılıysa `true`, başarısızsa `false` döndürmelidir (aksi hâlde `TypeError` oluşur).

Yeni değerleri doğrulamak için bunu kullanalım:

```js run
let numbers = [];

numbers = new Proxy(numbers, { // (*)
*!*
  set(target, prop, val) { // özelliğe yazmayı yakala
*/!*
    if (typeof val == 'number') {
      target[prop] = val;
      return true;
    } else {
      return false;
    }
  }
});

numbers.push(1);
numbers.push(2);
alert("Length is: " + numbers.length); // 2

*!*
numbers.push("test"); // TypeError ('set' on proxy returned false)
*/!*

alert("Bu satıra asla ulaşılmaz (yukarıdaki satırda hata var)");
```

Dikkat ederseniz, dizinin yerleşik işlevleri hâlâ çalışıyor!
Yeni değerler eklendiğinde `length` özelliği otomatik olarak artıyor. Proxy’miz hiçbir şeyi bozmadı.

Ayrıca`push`, `unshift` gibi değer ekleyen metotları da yeniden tanımlamamız gerekmedi.
Çünkü bunlar dahili olarak `[[Set]]` işlemini kullanırlar ve bu işlem proxy tarafından yakalanır.

Kod bu sayede hem temiz hem de kısa olur.

```warn header="`true` döndürmeyi unutmayın"
Yukarıda belirtildiği gibi, bazı değişmez kurallar (invariant) vardır.

`set` işlemi başarılıysa `true` döndürmelidir.

Eğer yanlış (falsy) bir değer döndürülürse (veya hiç değer döndürülmezse), bu `TypeError` hatasına neden olur.
```

## "deleteProperty" ve "ownKeys" ile korunan özellikler

Yaygın bir konvansiyona göre, `_` (alt çizgi) ile başlayan özellikler ve metotlar içseldir.  
Bu tür özelliklere nesnenin dışından erişilmemelidir.

Teknik olarak erişmek mümkündür:

```js run
let user = {
  name: "John",
  _password: "secret"
};

alert(user._password); // secret  
```

Şimdi `_` ile başlayan özelliklere erişimi engellemek için proxy kullanalım.

Bunun için şu tuzaklara ihtiyacımız var:
- `get` okuma sırasında hata fırlatmak için,
- `set` yazma sırasında hata fırlatmak için,
- `deleteProperty` silme sırasında hata fırlatmak için,
- `ownKeys` `_` ile başlayan özellikleri `for...in` döngüsü veya `Object.keys()` gibi işlemlerden gizlemek için.

Kod şu şekilde olur:

```js run
let user = {
  name: "John",
  _password: "***"
};

user = new Proxy(user, {
*!*
  get(target, prop) {
*/!*
    if (prop.startsWith('_')) {
      throw new Error("Erişim reddedildi");
    }
    let value = target[prop];
    return (typeof value === 'function') ? value.bind(target) : value; // (*)
  },
*!*
  set(target, prop, val) { // yazma işlemini yakala
*/!*
    if (prop.startsWith('_')) {
      throw new Error("Erişim reddedildi");
    } else {
      target[prop] = val;
    }
  },
*!*
  deleteProperty(target, prop) { // silme işlemini yakala
*/!*  
    if (prop.startsWith('_')) {
      throw new Error("Erişim reddedildi");
    } else {
      delete target[prop];
      return true;
    }
  },
*!*
  ownKeys(target) { // özellik listesini yakala
*/!*
    return Object.keys(target).filter(key => !key.startsWith('_'));
  }
});

// "get" -> _password okunamaz
try {
  alert(user._password); // Hata: Erişim reddedildi
} catch(e) { alert(e.message); }

// "set" -> _password yazılamaz
try {
  user._password = "test"; // Hata: Erişim reddedildi
} catch(e) { alert(e.message); }

// "deleteProperty" -> _password silinemez
try {
  delete user._password; // Hata: Erişim reddedildi
} catch(e) { alert(e.message); }

// "ownKeys" -> _password filtrelenir
for(let key in user) alert(key); // name
```

Please note the important detail in `get` trap, in the line `(*)`:

```js
get(target, prop) {
  // ...
  let value = target[prop];
*!*
  return (typeof value === 'function') ? value.bind(target) : value; // (*)
*/!*
}
```

If an object method is called, such as `user.checkPassword()`, it must be able to access `_password`:

```js
user = {
  // ...
  checkPassword(value) {
    // object method must be able to read _password
    return value === this._password;
  }
}
```

Normally, `user.checkPassword()` call gets proxied `user` as `this` (the object before dot becomes `this`), so when it tries to access `this._password`, the property protection kicks in and throws an error. So we bind it to `target` in the line `(*)`. Then all operations from that function directly reference the object, without any property protection.

That solution is not ideal, as the method may pass the unproxied object somewhere else, and then we'll get messed up: where's the original object, and where's the proxied one.

As an object may be proxied multiple times (multiple proxies may add different "tweaks" to the object), weird bugs may follow.

So, for complex objects with methods such proxy shouldn't be used.

```smart header="Private properties of a class"
Modern JavaScript engines natively support private properties in classes, prefixed with `#`. They are described in the chapter <info:private-protected-properties-methods>. No proxies required.

Such properties have their own issues though. In particular, they are not inherited.
```


## "In range" with "has" trap

Let's say we have a range object:

```js
let range = {
  start: 1,
  end: 10
};
```

We'd like to use "in" operator to check that a number is in `range`.

The "has" trap intercepts "in" calls: `has(target, property)`

- `target` -- is the target object, passed as the first argument to `new Proxy`,
- `property` -- property name

Here's the demo:

```js run
let range = {
  start: 1,
  end: 10
};

range = new Proxy(range, {
*!*
  has(target, prop) {
*/!*
    return prop >= target.start && prop <= target.end
  }
});

*!*
alert(5 in range); // true
alert(50 in range); // false
*/!*
```

A nice syntactic sugar, isn't it?

## Wrapping functions: "apply"

We can wrap a proxy around a function as well.

The `apply(target, thisArg, args)` trap handles calling a proxy as function:

- `target` is the target object,
- `thisArg` is the value of `this`.
- `args` is a list of arguments.

For example, let's recall `delay(f, ms)` decorator, that we did in the chapter <info:call-apply-decorators>.

In that chapter we did it without proxies. A call to `delay(f, ms)` would return a function that forwards all calls to `f` after `ms` milliseconds.

Here's the function-based implementation:

```js run
// no proxies, just a function wrapper
function delay(f, ms) {
  // return a wrapper that passes the call to f after the timeout
  return function() { // (*)
    setTimeout(() => f.apply(this, arguments), ms);
  };
}

function sayHi(user) {
  alert(`Hello, ${user}!`);
}

// now calls to sayHi will be delayed for 3 seconds
sayHi = delay(sayHi, 3000);

sayHi("John"); // Hello, John! (after 3 seconds)
```

As you can see, that mostly works. The wrapper function `(*)` performs the call after the timeout.

But a wrapper function does not forward property read/write operations or anything else. So if we have a property on the original function, we can't access it after wrapping:

```js run
function delay(f, ms) {
  return function() {
    setTimeout(() => f.apply(this, arguments), ms);
  };
}

function sayHi(user) {
  alert(`Hello, ${user}!`);
}

*!*
alert(sayHi.length); // 1 (function length is the arguments count)
*/!*

sayHi = delay(sayHi, 3000);

*!*
alert(sayHi.length); // 0 (wrapper has no arguments)
*/!*
```


`Proxy` is much more powerful, as it forwards everything to the target object.

Let's use `Proxy` instead of a wrapping function:

```js run
function delay(f, ms) {
  return new Proxy(f, {
    apply(target, thisArg, args) {
      setTimeout(() => target.apply(thisArg, args), ms);
    }
  });
}

function sayHi(user) {
  alert(`Hello, ${user}!`);
}

sayHi = delay(sayHi, 3000);

*!*
alert(sayHi.length); // 1 (*) proxy forwards "get length" operation to the target
*/!*

sayHi("John"); // Hello, John! (after 3 seconds)
```

The result is the same, but now not only calls, but all operations on the proxy are forwarded to the original function. So `sayHi.length` is returned correctly after the wrapping in the line `(*)`.

We've got a "richer" wrapper.

There exist other traps, but probably you've already got the idea.

## Reflect

The `Reflect` API was designed to work in tandem with `Proxy`.

For every internal object operation that can be trapped, there's a `Reflect` method. It has the same name and arguments as the trap, and can be used to forward the operation to an object.

For example:

```js run
let user = {
  name: "John",
};

user = new Proxy(user, {
  get(target, prop, receiver) {
    alert(`GET ${prop}`);
*!*
    return Reflect.get(target, prop, receiver); // (1)
*/!*
  },
  set(target, prop, val, receiver) {
    alert(`SET ${prop} TO ${val}`);
*!*
    return Reflect.set(target, prop, val, receiver); // (2)
*/!*
  }
});

let name = user.name; // GET name
user.name = "Pete"; // SET name TO Pete
```

- `Reflect.get` gets the property, like `target[prop]` that we used before.
- `Reflect.set` sets the property, like `target[prop] = value`, and also ensures the correct return value.

In most cases, we can do the same thing without `Reflect`. But we may miss some peculiar aspects.

Consider the following example, it doesn't use `Reflect` and doesn't work right.

We have a proxied user object and inherit from it, then use a getter:

```js run
let user = {
  _name: "Guest",
  get name() {
    return this._name;
  }
};

user = new Proxy(user, {
  get(target, prop, receiver) {
    return target[prop]; // (*)
  }
});


let admin = {
  __proto__: user,
  _name: "Admin"
};

*!*
// Expected: Admin
alert(admin.name); // Guest (?!?)
*/!*
```

As you can see, the result is incorrect! The `admin.name` is expected to be `"Admin"`, not `"Guest"`! Without the proxy, it would be `"Admin"`, looks like the proxying "broke" our object.

![](proxy-inherit.svg)

Why this happens? That's easy to understand if we explore what's going on during the call in the last line of the code.

1. There's no `name` property in `admin`, so `admin.name` call goes to `admin` prototype.
2. The prototype is the proxy, so its `get` trap intercepts the attempt to read `name`.
3. In the line `(*)` it returns `target[prop]`, but what is the `target`?
    - The `target`, the first argument of `get`, is always the object passed to `new Proxy`, the original `user`.
    - So, `target[prop]` invokes the getter `name` with `this=target=user`.
    - Hence the result is `"Guest"`.

How to fix it? That's what the `receiver`, the third argument of `get` is for! It holds the correct `this`. We just need to call `Reflect.get` to pass it on.

Here's the correct variant:

```js run
let user = {
  _name: "Guest",
  get name() {
    return this._name;
  }
};

user = new Proxy(user, {
  get(target, prop, receiver) {
*!*
    return Reflect.get(target, prop, receiver); // (*)
*/!*
  }
});


let admin = {
  __proto__: user,
  _name: "Admin"
};

*!*
alert(admin.name); // Admin
*/!*
```

Now the `receiver` holding the correct `this` is passed to getter by `Reflect.get` in the line `(*)`, so it works correctly.

We could also write the trap as:

```js
get(target, prop, receiver) {
  return Reflect.get(*!*...arguments*/!*);
}
```

`Reflect` calls are named exactly the same way as traps and accept the same arguments. They were specifically designed this way.

So, `return Reflect...` provides a safe no-brainer to forward the operation and make sure we don't forget anything related to that.

## Proxy limitations

Proxies are a great way to alter or tweak the behavior of the existing objects, including built-in ones, such as arrays.

Still, it's not perfect. There are limitations.

### Built-in objects: Internal slots

Many built-in objects, for example `Map`, `Set`, `Date`, `Promise` and others make use of so-called "internal slots".

These are like properties, but reserved for internal purposes. Built-in methods access them directly, not via `[[Get]]/[[Set]]` internal methods. So `Proxy` can't intercept that.

Who cares? They are internal anyway!

Well, here's the issue. After such built-in object gets proxied, the proxy doesn't have these internal slots, so built-in methods will fail.

For example:

```js run
let map = new Map();

let proxy = new Proxy(map, {});

*!*
proxy.set('test', 1); // Error
*/!*
```

An attempt to set a value into a proxied `Map` fails, for the reason related to its [internal implementation](https://tc39.es/ecma262/#sec-map.prototype.set).

Internally, a `Map` stores all data in its `[[MapData]]` internal slot. The proxy doesn't have such slot. The `set` method tries to access `this.[[MapData]]` internal property, but because `this=proxy`, can't find it in `proxy` and just fails.

Fortunately, there's a way to fix it:

```js run
let map = new Map();

let proxy = new Proxy(map, {
  get(target, prop, receiver) {
    let value = Reflect.get(...arguments);
*!*
    return typeof value == 'function' ? value.bind(target) : value;
*/!*
  }
});

proxy.set('test', 1);
alert(proxy.get('test')); // 1 (works!)
```

Now it works fine, because `get` trap binds function properties, such as `map.set`, to the target object (`map`) itself.

Unlike the previous example, the value of `this` inside `proxy.set(...)` will be not `proxy`, but the original `map`. So when the internal implementation of `set` tries to access `this.[[MapData]]` internal slot, it succeeds.

```smart header="`Array` has no internal slots"
A notable exception: built-in `Array` doesn't use internal slots. That's for historical reasons, as it appeared so long ago.

So there's no such problem when proxying an array.
```

### Private fields

The similar thing happens with private class fields.

For example, `getName()` method accesses the private `#name` property and breaks after proxying:

```js run
class User {
  #name = "Guest";

  getName() {
    return this.#name;
  }
}

let user = new User();

user = new Proxy(user, {});

*!*
alert(user.getName()); // Error
*/!*
```

The reason is that private fields are implemented using internal slots. JavaScript does not use `[[Get]]/[[Set]]` when accessing them.

In the call `user.getName()` the value of `this` is the proxied user, and it doesn't have the slot with private fields.

Once again, the solution with binding the method makes it work:

```js run
class User {
  #name = "Guest";

  getName() {
    return this.#name;
  }
}

let user = new User();

user = new Proxy(user, {
  get(target, prop, receiver) {
    let value = Reflect.get(...arguments);
    return typeof value == 'function' ? value.bind(target) : value;
  }
});

alert(user.getName()); // Guest
```

That said, the solution has drawbacks, explained previously: it exposes the original object to the method, potentially allowing it to be passed further and breaking other proxied functionality.

### Proxy != target

Proxy and the original object are different objects. That's natural, right?

So if we store the original object somewhere, and then proxy it, then things might break:

```js run
let allUsers = new Set();

class User {
  constructor(name) {
    this.name = name;
    allUsers.add(this);
  }
}

let user = new User("John");

alert(allUsers.has(user)); // true

user = new Proxy(user, {});

*!*
alert(allUsers.has(user)); // false
*/!*
```

As we can see, after proxying we can't find `user` in the set `allUsers`, because the proxy is a different object.

```warn header="Proxies can't intercept a strict equality test `===`"
Proxies can intercept many operators, such as `new` (with `construct`), `in` (with `has`), `delete` (with `deleteProperty`) and so on.

But there's no way to intercept a strict equality test for objects. An object is strictly equal to itself only, and no other value.

So all operations and built-in classes that compare objects for equality will differentiate between the object and the proxy. No transparent replacement here.
```


## Revocable proxies

A *revocable* proxy is a proxy that can be disabled.

Let's say we have a resource, and would like to close access to it any moment.

What we can do is to wrap it into a revocable proxy, without any traps. Such proxy will forward operations to object, and we also get a special method to disable it.

The syntax is:

```js
let {proxy, revoke} = Proxy.revocable(target, handler)
```

The call returns an object with the `proxy` and `revoke` function to disable it.

Here's an example:

```js run
let object = {
  data: "Valuable data"
};

let {proxy, revoke} = Proxy.revocable(object, {});

// pass the proxy somewhere instead of object...
alert(proxy.data); // Valuable data

// later in our code
revoke();

// the proxy isn't working any more (revoked)
alert(proxy.data); // Error
```

A call to `revoke()` removes all internal references to the target object from the proxy, so they are no more connected. The target object can be garbage-collected after that.

We can also store `revoke` in a `WeakMap`, to be able to easily find it by the proxy:


```js run
*!*
let revokes = new WeakMap();
*/!*

let object = {
  data: "Valuable data"
};

let {proxy, revoke} = Proxy.revocable(object, {});

revokes.set(proxy, revoke);

// ..later in our code..
revoke = revokes.get(proxy);
revoke();

alert(proxy.data); // Error (revoked)
```

The benefit of such approach is that we don't have to carry `revoke` around. We can get it from the map by `proxy` when needeed.

Using `WeakMap` instead of `Map` here, because it should not block garbage collection. If a proxy object becomes "unreachable" (e.g. no variable references it any more), `WeakMap` allows it to be wiped from memory (we don't need its revoke in that case).

## References

- Specification: [Proxy](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots).
- MDN: [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy).

## Summary

`Proxy` is a wrapper around an object, that forwards operations to the object, optionally trapping some of them.

It can wrap any kind of object, including classes and functions.

The syntax is:

```js
let proxy = new Proxy(target, {
  /* traps */
});
```

...Then we should use `proxy` everywhere instead of `target`. A proxy doesn't have its own properties or methods. It traps an operation if the trap is provided or forwards it to `target` object.

We can trap:
- Reading (`get`), writing (`set`), deleting (`deleteProperty`) a property (even a non-existing one).
- Calling functions with `new` (`construct` trap) and without `new` (`apply` trap)
- Many other operations (the full list is at the beginning of the article and in the [docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)).

That allows us to create "virtual" properties and methods, implement default values, observable objects, function decorators and so much more.

We can also wrap an object multiple times in different proxies, decorating it with various aspects of functionality.

The [Reflect](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect) API is designed to complement [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy). For any `Proxy` trap, there's a `Reflect` call with same arguments. We should use those to forward calls to target objects.

Proxies have some limitations:

- Built-in objects have "internal slots", access to those can't be proxied. See the workaround above.
- The same holds true for private class fields, as they are internally implemented using slots. So proxied method calls must have the target object as `this` to access them.
- Object equality tests `===` can't be intercepted.
- Performance: benchmarks depend on an engine, but generally accessing a property using a simplest proxy takes a few times longer. In practice that only matters for some "bottleneck" objects though.
