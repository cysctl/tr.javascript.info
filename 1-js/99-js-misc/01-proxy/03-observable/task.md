
# Gözlemlenebilir (Observable)

Bir nesneyi "gözlemlenebilir" yapan ve bir proxy döndüren `makeObservable(target)` fonksiyonunu oluşturun.

Şöyle çalışmalı:

```js run
function makeObservable(target) {
  /* kodunuz */
}

let user = {};
user = makeObservable(user);

user.observe((key, value) => {
  alert(`SET ${key}=${value}`);
});

user.name = "John"; // alerts: SET name=John
```

Başka bir deyişle, `makeObservable` tarafından döndürülen nesnede `observe(handler)` metodu bulunur.

Bir özelliğin değeri değiştiğinde, ilgili özelliğin adı ve değeri ile `handler(key, value)` çağrılır.

Not: Bu görevde yalnızca bir özelliğe değer atamayı (yazmayı) ele alın. Diğer işlemler benzer şekilde uygulanabilir.
Ek Not: Handler'ları saklamak için global bir değişken veya global bir yapı kullanabilirsiniz. Burada bu uygundur. Gerçek hayatta, böyle bir fonksiyon kendi global kapsamına sahip bir modülde yaşar.
