
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

Lütfen `get` tuzağındaki, `(*)` satırındaki önemli detaya dikkat edin:

```js
get(target, prop) {
  // ...
  let value = target[prop];
*!*
  return (typeof value === 'function') ? value.bind(target) : value; // (*)
*/!*
}
```

Bir nesne metodu çağrıldığında, örneğin `user.checkPassword()`, bu metodun `_password`’a erişebilmesi gerekir:

```js
user = {
  // ...
  checkPassword(value) {
    // nesne metodu _password'ı okuyabilmeli
    return value === this._password;
  }
}
```

Normalde, `user.checkPassword()` çağrısında `this` olarak proxylanmış `user` geçer (noktadan önceki nesne `this` olur). Bu yüzden metod `this._password`’a erişmeye çalıştığında, özellik koruması devreye girer ve hata fırlatır. İşte bu nedenle `(*)` satırında metodu `target`’a bağlarız (`bind`). Böylece o fonksiyon içindeki tüm işlemler doğrudan orijinal nesneye yapılır ve özellik korumasına takılmaz.

Bu çözüm ideal değildir; çünkü metot, proxylanmamış nesneyi başka bir yere aktarabilir ve sonrasında işler karışabilir: Orijinal nesne nerede, proxy nerede?

Bir nesne birden fazla kez proxylanabilir (farklı proxy’ler nesneye farklı “ince ayarlar” ekleyebilir), bu da garip hatalara yol açabilir.

Dolayısıyla, metotları olan karmaşık nesneler için bu tür bir proxy kullanımı önerilmez.

```smart header="Private properties of a class"
Modern JavaScript motorları, `#` ile başlayan sınıf içi özel özellikleri yerel olarak destekler. Bunlar <info:private-protected-properties-methods> bölümünde anlatılmıştır. Proxy gerekmez.

Ancak bu özelliklerin de kendine özgü sorunları vardır. Özellikle, kalıtılmazlar.
```


## "has" tuzağı ile "in range" (aralıkta mı?) denetimi

Bir aralık (range) nesnemiz olduğunu varsayalım:

```js
let range = {
  start: 1,
  end: 10
};
```

Bir sayının `range` içinde olup olmadığını denetlemek için "in" operatörünü kullanmak istiyoruz.

"has" tuzağı, "in" çağrılarını yakalar: `has(target, property)`

- `target` -- `new Proxy`’ye ilk argüman olarak geçirilen hedef nesne,,
- `property` -- özellik adı

Örnek:

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

Güzel bir sözdizim şekeri, değil mi?

## Fonksiyonları sarmalamak: "apply"

Bir fonksiyonun etrafına da proxy sarabiliriz.

`apply(target, thisArg, args)` tuzağı, proxy’nin fonksiyon gibi çağrılmasını yakalar:

- `target` hedef nesne (fonksiyon),
- `thisArg` çağrıda kullanılacak `this` değeri,
- `args` argümanların listesi.

Örneğin, <info:call-apply-decorators> bölümünde yaptığımız `delay(f, ms)` dekoratörünü hatırlayalım.

Orada proxy kullanmadan yapmıştık. `delay(f, ms)` çağrısı, tüm çağrıları `ms` milisaniye sonra `f`’e ileten bir fonksiyon döndürüyordu.

Fonksiyon-tabanlı uygulama:

```js run
// proxy yok, sadece bir sarmalayıcı fonksiyon
function delay(f, ms) {
  // timeout sonrası çağrıyı f'ye ileten bir sarmalayıcı döndür
  return function() { // (*)
    setTimeout(() => f.apply(this, arguments), ms);
  };
}

function sayHi(user) {
  alert(`Hello, ${user}!`);
}

// artık sayHi çağrıları 3 saniye gecikmeli
sayHi = delay(sayHi, 3000);

sayHi("John"); // Hello, John! (3 saniye sonra)
```

Gördüğünüz gibi, çoğunlukla çalışıyor. `(*)` satırındaki sarmalayıcı fonksiyon, çağrıyı bekleme süresinden sonra gerçekleştiriyor.

Ama sarmalayıcı fonksiyon, özellik okuma/yazma gibi diğer işlemleri iletmez. Dolayısıyla, orijinal fonksiyonun bir özelliği varsa, sarmalamadan sonra ona erişemeyiz:

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
alert(sayHi.length); // 1 (function length argüman sayısıdır)
*/!*

sayHi = delay(sayHi, 3000);

*!*
alert(sayHi.length); // 0 (sarmalayıcının argümanı yok)
*/!*
```


`Proxy` çok daha güçlüdür; çünkü her şeyi hedef nesneye iletir.

Sarmalayıcı fonksiyon yerine `Proxy` kullanalım:

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
alert(sayHi.length); // 1 (*) proxy "length" okuma işlemini hedefe iletir
*/!*

sayHi("John"); // Hello, John! (3 saniye sonra)
```

Sonuç aynı, ancak artık yalnızca çağrılar değil, proxy üzerindeki tüm işlemler de orijinal fonksiyona iletiliyor. Bu nedenle `(*)` satırında sarmalamadan sonra `sayHi.length` doğru şekilde döndü.

Böylece daha “zengin” bir sarmalayıcı elde etmiş olduk.

Başka trap’ler de var, fakat sanırım artık mantığı anladınız.

## Reflect

`Reflect` API’si, `Proxy` ile birlikte çalışmak üzere tasarlanmıştır.

Yakalanabilecek (trap’lenebilecek) her dahili nesne işlemi için bir `Reflect` metodu vardır.  
Bu metodun adı ve parametreleri, ilgili trap ile aynıdır ve işlemi hedef nesneye iletmek için kullanılabilir.


Örneğin:

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

- `Reflect.get`, tıpkı `target[prop]` gibi özelliği okur.
- `Reflect.set`, tıpkı `target[prop] = value` gibi özelliği yazar, ayrıca doğru dönüş değerini de garanti eder.

Çoğu durumda `Reflect` kullanmadan da aynı işi yapabiliriz, ancak bazı ince detayları kaçırabiliriz.

Aşağıdaki örneğe bakalım; `Reflect` kullanılmamış ve yanlış sonuç veriyor:

Bir user nesnesini proxy’liyoruz, sonra ondan kalıtım alıp bir getter kullanıyoruz:

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
// Beklenen: Admin
alert(admin.name); // Guest (?!?)
*/!*
```

Gördüğünüz gibi sonuç hatalı! `admin.name` değerinin `"Admin"` olması gerekirken `"Guest"` döndü. Proxy olmadan `"Admin"` olacaktı; proxyleme nesneyi “bozmuş” gibi görünüyor.

![](proxy-inherit.svg)

Peki neden? Son satırdaki çağrının ne yaptığına bakalım:

1. `admin` içinde `name` özelliği yok, bu yüzden çağrı `admin`’in prototipine gider.
2. Prototip bir proxy olduğu için, `get` tuzağı `name` okuma girişimini yakalar.
3. `(*)` satırında `target[prop]` döndürülür, peki `target` nedir?
    - `target`, `get`’in ilk parametresidir ve her zaman `new Proxy`’ye verilen orijinal nesnedir(`user`).
    - Dolayısıyla, `target[prop]` getter `name`’i `this=target=user` olarak çağırır.
    - Bu yüzden sonuç `"Guest"` olur.

Bunu nasıl düzeltiriz? İşte bu noktada üçüncü parametre olan `receiver` devreye girer! `receiver`, doğru `this` değerini tutar. Sadece `Reflect.get`’i çağırarak bu değeri aktarabiliriz.

Doğru çözüm şöyle olur:

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

Artık `receiver`, doğru `this` değerini tutarak getter’a `Reflect.get` aracılığıyla (`(*)` satırında) aktarılır ve her şey düzgün çalışır.

Ayrıca tuzağı şu şekilde de yazabilirdik:

```js
get(target, prop, receiver) {
  return Reflect.get(*!*...arguments*/!*);
}
```

`Reflect` çağrıları, tuzaklarla birebir aynı isimlere ve argümanlara sahiptir.
Bu özellikle böyle tasarlanmıştır.

Dolayısıyla `return Reflect...` şeklinde bir çağrı, işlemi güvenli bir şekilde iletmenin ve hiçbir detayı unutmamanın en kolay yoludur.

## Proxy Sınırlamaları

Proxy’ler, var olan nesnelerin (hatta yerleşik olanların, örn. diziler) davranışlarını değiştirmek veya özelleştirmek için mükemmel bir araçtır.

Yine de, bazı sınırlamaları vardır.

### Yerleşik Nesneler: İçsel Slotlar (Internal Slots)

Birçok yerleşik nesne, örneğin `Map`, `Set`, `Date`, `Promise` ve benzerleri, “içsel slot” (internal slot) denen yapıları kullanır.

Bunlar özelliklere benzer, ancak dahili amaçlar için ayrılmıştır.
Yerleşik metotlar bu slotlara doğrudan erişir, `[[Get]]/[[Set]]` gibi içsel metotları kullanmazlar. Bu yüzden `Proxy` bunları yakalayamaz.

“Eee ne olmuş, sonuçta dahili yapılar!” diyebilirsiniz.

Ama sorun şudur:
Böyle bir yerleşik nesne proxylenirse, proxy bu içsel slotlara sahip olmaz ve dolayısıyla yerleşik metotlar çalışmaz.

Örneğin:

```js run
let map = new Map();

let proxy = new Proxy(map, {});

*!*
proxy.set('test', 1); // Hata
*/!*
```

Bir `Map` üzerinde `set` çağırmak başarısız olur; çünkü bu davranış [içsel implementasyona](https://tc39.es/ecma262/#sec-map.prototype.set) bağlıdır.

Dahili olarak bir `Map`, tüm verilerini `[[MapData]]` adlı içsel slotta saklar. Proxy’de böyle bir slot yoktur. `set` metodu `this.[[MapData]]`’ya erişmeye çalışır; ancak `this=proxy` olduğu için bulamaz ve hata verir.

Neyse ki bunu düzeltmenin bir yolu var:

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
alert(proxy.get('test')); // 1 (çalışıyor!)
```

Şimdi düzgün çalışıyor, çünkü `get` tuzağı fonksiyon özellikleri (`map.set` gibi), hedef nesneye (`map`) bağlıyor.

Böylece `proxy.set(...)` içindeki `this` artık `proxy` değil, orijinal `map` olur. Dolayısıyla `set` metodu `this.[[MapData]]` slotuna erişmeye çalıştığında başarılı olur.

```smart header="`Array`'in içsel slotları yoktur"
Belirgin bir istisna: yerleşik `Array` içsel slotlar kullanmaz. Bu tarihsel bir sebepten dolayıdır, çok eski bir yapı olduğundan.

Bu nedenle, dizileri proxy’lemek bu tür bir soruna yol açmaz.
```

### Özel Alanlar (Private Fields)

Benzer bir durum, sınıflardaki özel (private) alanlarda da görülür.

Örneğin, `getName()` metodu özel `#name` alanına erişir, ancak proxy uygulandıktan sonra bozulur:

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

Bunun nedeni, özel alanların (private fields) dahili slotlar (internal slots) kullanılarak uygulanmasıdır.
JavaScript, bunlara erişirken `[[Get]]` veya `[[Set]]` mekanizmalarını kullanmaz.

`user.getName()` çağrısında, `this` değeri proxylanmış kullanıcı nesnesidir ve bu nesnede özel alanların bulunduğu slot yoktur.

Daha önceki örneklerde olduğu gibi, metodu hedef nesneye bağlayarak (`bind`) bu durumu düzeltebiliriz:

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

Ancak bu çözümün daha önce açıkladığımız bir dezavantajı var:
Metot, orijinal nesneye doğrudan erişim kazanır ve onu başka bir yere aktarabilir.
Bu da diğer proxy işlevselliklerini bozabilir.

### Proxy != target

Proxy ve orijinal nesne farklı nesnelerdir. Bu oldukça doğaldır, değil mi?

Yani bir nesneyi bir yerde saklayıp daha sonra proxy’larsak, bazı şeyler bozulabilir:

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

Gördüğünüz gibi, proxylamadan sonra `user`, `allUsers` kümesinde bulunamıyor çünkü proxy farklı bir nesne.

```warn header="Proxy'ler sıkı eşitlik testini `===` yakalayamaz"
Proxy’ler `new` (-> `construct`), `in` (-> `has`), `delete` (-> `deleteProperty`) gibi birçok işlemi yakalayabilir.

Ancak, nesneler için sıkı eşitlik testi (`===`) yakalanamaz.
Bir nesne yalnızca kendisine eşittir; başka hiçbir şeye değil.

Bu nedenle, nesneleri karşılaştıran tüm işlemler ve yerleşik sınıflar, nesne ile proxy arasında ayrım yapar.
Yani proxy’ler tam anlamıyla “şeffaf” bir yedek olamaz.
```


## Geri Alınabilir Proxy’ler (Revocable Proxies)

*Revocable proxy*, yani “geri alınabilir proxy”, devre dışı bırakılabilen bir proxy türüdür.

Diyelim ki bir kaynağımız var ve istediğimiz anda bu kaynağa erişimi kapatmak istiyoruz.

Bunu, herhangi bir trap eklemeden bir “revocable proxy” ile sarmalayabiliriz.  
Bu proxy işlemleri hedef nesneye iletir, ancak ayrıca onu devre dışı bırakmak için özel bir fonksiyon sağlar.


Sözdizimi şöyledir:

```js
let {proxy, revoke} = Proxy.revocable(target, handler)
```

Bu çağrı, iki özellik döndürür:
- `proxy` vekil nesne,
- `revoke` bu proxy’yi devre dışı bırakmak için kullanılan fonksiyon.

Örnek:

```js run
let object = {
  data: "Valuable data"
};

let {proxy, revoke} = Proxy.revocable(object, {});

// nesne yerine proxy’yi bir yere aktarabiliriz...
alert(proxy.data); // Valuable data

// kodun ilerleyen bir kısmında
revoke();

// proxy artık çalışmaz (devre dışı)
alert(proxy.data); // Hata
```

`revoke()` çağrısı, proxy’nin hedef nesneyle olan tüm dahili bağlantılarını kaldırır.
Böylece artık birbirleriyle ilişkili değildirler.
Bu noktadan sonra hedef nesne, çöp toplayıcı (garbage collector) tarafından silinebilir.

Ayrıca `revoke` fonksiyonunu bir `WeakMap` içinde saklayarak, proxy üzerinden kolayca bulabiliriz:


```js run
*!*
let revokes = new WeakMap();
*/!*

let object = {
  data: "Valuable data"
};

let {proxy, revoke} = Proxy.revocable(object, {});

revokes.set(proxy, revoke);

// ...kodun ilerleyen kısmında...
revoke = revokes.get(proxy);
revoke();

alert(proxy.data); // Hata (revoked)
```

Bu yaklaşımın avantajı, `revoke` fonksiyonunu her yere taşımak zorunda olmamamızdır.
Proxy üzerinden `WeakMap` aracılığıyla gerektiğinde erişebiliriz.

Burada `Map` yerine `WeakMap` kullanmamızın nedeni, çöp toplamayı engellememesidir.
Eğer bir proxy nesnesine artık erişilmiyorsa (örneğin hiçbir değişken onu tutmuyorsa), `WeakMap` onun bellekten silinmesine izin verir. Çünkü artık `revoke` fonksiyonuna da ihtiyacımız yoktur.

## Referanslar

- Spesifikasyon: [Proxy](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots).
- MDN: [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy).

## Özet

`Proxy`, bir nesnenin etrafını saran ve işlemleri o nesneye yönlendiren (ve istenirse bazılarını yakalayan) bir yapıdır.

Her tür nesneyi, sınıflar ve fonksiyonlar dahil, sarmalayabilir.

Sözdizimi:

```js
let proxy = new Proxy(target, {
  /* tuzaklar */
});
```

...Bundan sonra `target` yerine her yerde `proxy` kullanılmalıdır. Bir proxy’nin kendi özellikleri veya metotları yoktur.
Bir işlem, uygun bir trap tanımlanmışsa yakalanır; yoksa hedef nesneye (`target`) iletilir.

Yakalanabilecek işlemler şunlardır:
- Özellik okuma (`get`), yazma (`set`), silme (`deleteProperty`), hatta var olmayan bir özellik bile.
- Fonksiyon çağrıları: `new` ile (`construct` tuzağı), `new` olmadan (`apply` tuzağı).
- Daha birçok işlem (tam liste makalenin başında ve [MDN dokümentasyonunda](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) yer alır).

Bu sayede:
- “Sanal” (virtual) özellikler ve metotlar oluşturabiliriz,
- Varsayılan değerler tanımlayabiliriz,
- Gözlemlenebilir (observable) nesneler oluşturabiliriz,
- Fonksiyon dekoratörleri (decorators) yazabiliriz,
ve çok daha fazlasını yapabiliriz.


Bir nesneyi farklı `Proxy` katmanlarıyla birden fazla kez sarmalayabiliriz. Her biri farklı bir işlevsellik ekleyebilir.

[Reflect](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect) API'si [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)’yi tamamlamak üzere tasarlanmıştır. Her `Proxy` tuzağı için aynı argümanları alan bir `Reflect` çağrısı bulunur. Bu çağrılar, işlemleri hedef nesneye güvenli bir şekilde iletmek için kullanılmalıdır.

Proxy’lerin bazı sınırlamaları vardır:

- Yerleşik nesneler, "içsel slot" (internal slot) adı verilen alanlara sahiptir; bunlara erişim proxy ile yakalanamaz. (Yukarıda çözüm örneği verilmiştir.)
- Aynı durum özel (private) sınıf alanları için de geçerlidir; bunlar da slot’lar aracılığıyla uygulanır. Bu nedenle proxylanmış metod çağrıları, bu alanlara erişebilmek için `this` değerinin hedef nesne olması gerekir.
- Nesne eşitlik testi (`===`) yakalanamaz.
- Performans: Proxy üzerinden özellik erişimi genellikle birkaç kat daha yavaştır. Ancak pratikte bu fark yalnızca dar boğaz oluşturan nesnelerde hissedilir.
