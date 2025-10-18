
# Mevcut olmayan özelliği okuma hatası

Mevcut olmayan bir özelliği okumaya çalışıldığında hata fırlatan bir proxy oluşturun.

Bu, programlama hatalarını erken tespit etmeye yardımcı olabilir.

Bir nesne `target` alan ve bu işlevselliğe sahip bir proxy döndüren `wrap(target)` fonksiyonunu yazın.
Şöyle çalışmalı:

```js
let user = {
  name: "John"
};

function wrap(target) {
  return new Proxy(target, {
*!*
      /* kodunuz */
*/!*
  });
}

user = wrap(user);

alert(user.name); // John
*!*
alert(user.age); // Hata: Özellik yok
*/!*
```
