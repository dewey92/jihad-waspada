---
title: "Type Class Dan Cara Kerjanya Di Balik Layar"
date: 2019-08-22T23:01:35+02:00
description: "Type Class adalah sebuah cara untuk memberikan instance dictionary secara implisit"
images: ["/uploads/gear.jpg"]
image:
  src: "/uploads/gear.jpg"
  caption: Image by <a href="https://pixabay.com/users/JarkkoManty-661512/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2291916">Jarkko Mänty</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2291916">Pixabay</a>
tags: ["purescript", "types", "typeclass"]
categories: ["programming", "purescript", "type system"]
draft: false
---

Buat kamu-kamu yang punya background di OOP dan udah [kenalan sama konsep interface](https://medium.com/@Dewey92/oop-interface-what-ca16de0359af), pasti enak banget kan bisa bikin aplikasi yang polymorphic dan gak terikat sama specific implementation. Kalau mau nambah variant, tinggal bikin class baru yang implement interface tersebut, selesai deh gak perlu ubah-ubah bagian code yang lain.

Klo interface di FP gimana, pak? Tergantung bahasanya, karena di FP ada bahasa yang typed dan untyped. Berhubung konsep interface ini ada di bahasa yang typed dan saya lagi explore Purescript sebagai bahasa FP yang typed, bolehlah saya bantu jawab sedikit bahwasanya konsep interface ini bisa disandarkan dengan "type class". Class di Purescript nggak sama kayak class di bahasa OOP pada umumnya, class di PS justru lebih kayak interface. More on this later.

Saya ambil contoh penggunaan interface di Typescript.

```ts
interface Size {
  len: () => number;
  whoAreYou: () => string;
}

class MyNumber implements Size {
  constructor(private num: number) {}

  len = () => this.num
  whoAreYou = () => "aku number"
}

class MyString implements Size {
  constructor(private str: string) {}

  len = () => this.str.length
  whoAreYou = () => "aku string"
}
```

Daaan kita bisa buat sebuah function yang polymorphic terhadap segala jenis `Size`:

```ts
const lengthPlusTen = (size: Size) => size.len() + 10

const res1 = lengthPlusTen(new MyNumber(5))         // 15
const res2 = lengthPlusTen(new MyString('waspada')) // 17
```

Fungsi `lengthPlusTen` jelas dapat mengakses method `len()` karena variable `size` merupakan **instance dari `Size`** yang memiliki method `len()`. Sekarang bandingkan dengan "interface" yang ada di Purescript.

```hs
class Size s where
  len :: s -> Int
  whoAreYou :: s -> String

instance sizeInt :: Size Int where
  len n = n
  whoAreYou _ = "aku integer"

instance sizeStr :: Size String where
  len = Data.String.length
  whoAreYou _ = "aku string"

lengthPlusTen :: ∀ s. Size s => s -> Int
lengthPlusTen s = (len s) + 10

res1 = lengthPlusTen 5         -- 15
res2 = lengthPlusTen "waspada" -- 17
```

Di sini kita nggak mengirim **instance** apa-apa seperti yang kita lakukan di Typescript, alih-alih hanya mengirimkan plain integer (`5`) dan plain string (`"waspada"`). Namun compiler automagically bisa dengan benar memilih instance mana yang harus digunakan untuk masing-masing data.

Muncul pertanyaan: sebenernya gimana sih cara kerja "class" ini di balik layar sehingga compiler bisa tahu mana instance yang harus diambil?

## Dictionary
Saya sendiri sebenarnya belum tahu **secara persis** bagaimana proses class resolution dilakukan di Purescript. Namun saya menemukan [artikel menarik](https://www.schoolofhaskell.com/user/jfischoff/instances-and-dictionaries) yang menjelaskan bagaimana Haskell menggunakan teknik _dictionary passing_ agar compiler dapat menentukan dan memilih instance yang benar.

Saya pun double-check di Slack Channel <mark>#purescript</mark> apakah proses tersebut juga dilakukan oleh compiler Purescript, Pak Harry menjawab iya. Mantaplah qlo beqitu 🎉🎉🎉. Dengan catatan, hanya rough idea aja ya 🙂

Jadi gini bapak-bapak ibu-ibu, setiap class yang kita buat, compiler juga membuat representasinya sendiri dengan sesuatu yang disebut "dictionary". Misal untuk contoh kasus `class Size n` tadi, compiler akan melakukan proses _desugaring_ untuk class tersebut dan membuat dictionary-nya dengan record.

```hs
type SizeDict s = {
  len :: s -> Int,
  whoAreYou :: s -> String
}
```

Juga untuk tiap-tiap "instance"-nya. Saya ambil yang integer saja dulu.

```hs
sizeIntDict :: SizeDict Int
sizeIntDict = {
  len: \n -> n,
  whoAreYou: \_ -> "aku integer"
}
```

Pun function `len` yang semula memiliki type signature `∀ s. Size s => s -> Int` juga mengalami proses _desugaring_ ini.

```hs
-- semula
len     :: ∀ s. Size     s => s -> Int

-- menjadi
lenDict :: ∀ s. SizeDict s -> s -> Int
lenDict dict s = dict.len s
```

Eaa sekarang malah mirip kayak contoh Typescript tadi!

Dengan proses _desugaring_ seperti ini, ketika saya menyuplai fungsi `len` dengan angka `5`, maka sebenarnya code saya akan "diubah" menjadi:

```hs
-- semula
res1 = len 5
-- menjadi
res1 = lenDict ? 5
```

Somehow kita membutuhkan `sizeIntDict` untuk mengisi placeholder `?`. Dan kita hanya bisa mengandalkan compiler untuk melakukan type inference pada fungsi `lenDict` yang mana (dalam konteks ini) pasti memiliki type signature:

```hs
lenDict :: SizeDict Int -> Int -> Int
```

Setelah [monotype](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system#Monotypes) diketahui (`Int`), compiler gak menemukan kandidat yang type signature-nya cocok dengan `SizeDict Int` selain variable `sizeIntDict`. Akhirnya compiler pun menyuplai dictionary ini ke fungsi `lenDict`:

```hs
res1 = lenDict sizeIntDict 5
```

Dan selesailah proses "pencarian instance" ini dengan teknik _dictionary passing_. Seluruh proses ini memberikan kesimpulan bahwa type class adalah sebuah cara untuk memberikan instance dictionary **secara implisit**.

## Compile ke Javascript
"Sisa-sisa" proses desugaring dengan dictionary ini bisa dilihat ketika code di atas di-compile ke Javascript.

```js
// class Size s ...
var Size = function (len, whoAreyou) {
  this.len = len;
  this.whoAreyou = whoAreyou;
};

var len = function (dict) { // `lenDict`
  return dict.len;
};
var whoAreyou = function (dict) {
  return dict.whoAreyou;
};

// instance sizeInt :: Size Int ...
var sizeInt = new Size( // `sizeIntDict`
  function (n) {
    return n;
  },
  function (_) {
    return "aku integer";
  }
);

// instance sizeStr :: Size String ...
var sizeStr = new Size(
  Data_String.length,
  function (_) {
    return "aku string";
  }
);

var lengthPlusTen = function (dictSize) {
  return function (s) {
    return len(dictSize)(s) + 10 | 0;
  };
};
var res1 = lengthPlusTen(sizeInt)(5);
                         ^^^^^^^
//                  dictionary integer
```

Dari output ini terlihat jelas bahwa fungsi `len`, `whoAreYou`, dan fungsi lain yang depend on them seperti `lengthPlusTen` membutuhkan "dictionary" saat runtime. Dictionary tersebut di-resolve oleh compiler saat compile time bergantung pada konteksnya. Artinya kalo kita kasih string ke dalam fungsi `len`, dictionary yang di-resolve oleh compiler pun pasti akan berbeda.

```hs
res2 = len "waspada"
```

```js
var res2 = len(sizeStr)("waspada");
               ^^^^^^^
//        dictionary string
```

---

Sekian dari saya kurang lebihnya diambil aja 🙏🏻

_Wa billahi ttaufiq walhidayah, wa rridho wal'inayah_, wa dzikri waspada.
Wassalamu 'alaikum warahmatullahi wabarakatuh.