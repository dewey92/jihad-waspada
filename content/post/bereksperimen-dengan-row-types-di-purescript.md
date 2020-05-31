---
title: "Bereksperimen dengan Row Types di Purescript"
date: 2019-08-21T15:20:12+02:00
description: "Fokus artikel ini lebih ke type-level programming untuk Row Types di Purescript. Saya mencoba menjelaskan bagaimana membuat type signature yang agak kompleks step by step"
images: ["/uploads/row-houses.jpg"]
image:
  src: "/uploads/row-houses.jpg"
  caption: Image by <a href="https://pixabay.com/users/12019-12019/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2220459">David Mark</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2220459">Pixabay</a>
tags: ["purescript", "typesystem", "rowpolymorphism"]
categories: ["programming", "purescript", "type system"]
draft: false
---

Belakangan saya lagi iseng-iseng main metaprogramming sama record system-nya Purescript, gimana cara nambahin value, ngubah type suatu attribute (key), menghilangkan attribute, dan lain-lain. Basic idea-nya sama kayak artikel saya yang lalu soal Row Polymorphism di Typescript. Tapi kali ini pake Purescript: gimana caranya operasi-operasi tersebut dapat ditangkap dan di-infer oleh compiler biar program tetep safe dan typechecked.

Artikel ini gak akan berfokus pada akses record di Purescript, tapi lebih ke arah type-level programming untuk manipulasi record. Jadi contoh-contoh di bawah nanti gak akan saya sertakan implementation details-nya, cuman type signature aja 😁

**NOTE**: Kalau temen-temen menemukan kata "attribute", "label", atau "key" di artikel ini, perlu saya garisbawahi bahwa saya merujuk pada hal yang sama :))

## Symbol
Sebelum masuk ke pembahasan yang lebih lanjut, kita harus mengetahui terlebih dahulu apa maksud Symbol di Purescript. Symbol yang ini jangan dibayangkan kayak Symbol yang ada di Javascript ya 😅 Symbol di Puresctipt digunakan untuk merepresentasikan suatu string yang beroperasi **di type level** dan bukan di term level. Artinya string di level type dan string di level term adalah dua hal yang berbeda yang hidup di alam yang berbeda juga 👻

Nah `SProxy` (String Proxy) dapat membantu kita untuk "mengangkat derajat" sebuah string dari term level ke type level (dari String menjadi Symbol) dan function `reflectSymbol` untuk "menurunkan derajat" dari type level string kembali ke term level string.

```hs
termLevel = "Jihad"

typeLevel :: SProxy "Jihad"
typeLevel = Sproxy
-- atau
typeLevel = (SProxy :: _ "Jihad")

jihad = reflectSymbol typeLevel
jihad == termLevel
```

{{< figure src="https://media.giphy.com/media/12NUbkX6p4xOO4/giphy.gif" alt="one on one" caption="Compiler magic" class="fig-center img-60" >}}

Oh ya satu lagi, Purescript juga menyediakan class `IsSymbol` yang bisa digunakan sebagai constraint untuk meng-assert (apa ya bahasa indonya?) suatu type variable sebagai Symbol.

```hs
acceptsAndReturnsProxy :: ∀ a.
  IsSymbol a =>
  SProxy a -> SProxy a
acceptsAndReturnsProxy = identity
```

Pada section berikutnya kita akan melihat kombinasi antara `IsSymbol` dan `SProxy` banyak digunakan untuk merujuk pada suatu attribute record di level type.

## Cons
Sewaktu artikel ini ditulis (`v13.3`), Purescript memiliki beberapa utility class untuk dapat memanipulasi dan menangkap informasi record saat compile-time. Salah satu yang paling sering digunakan adalah `Cons`.

```hs
-- module Prim.Row

class Cons (label :: Symbol) (a :: Type) (tail :: # Type) (row :: # Type)
  | label a tail -> row
  , label row -> a tail
```

class `Cons` mengekspresikan **sebuah record yang memiliki attribute `label` yang bertipe `a`**. `tail` adalah sisa `row` tanpa `label`.

Perlu diperhatikan bahwa row bisa memiliki label yang duplikat di type level. Saya udah nanya soal kenapa row types bisa mengandung duplikasi label di slack channel tapi pak Harry sendiri juga gatau kenapa haha. Yo-uwes, mungkin kapan-kapan dijawab sama yang lebih berwenang.

Anyway, `Cons` ini bisa diibaratkan seperti ini:

```hs
class Cons (label :: Symbol) (a :: Type) (tail :: # Type) (row :: # Type)
  | label a tail -> row
  , label row -> a tail

-- `tail` is inferred as `()`
Cons "name" String tail (name :: String)

-- `tail` is inferred as `(name :: String)`
Cons "age" Int tail (name :: String, age :: Int)

-- Duplicate label "name"
-- `tail` is inferred as `(age :: Int, name :: Array String)`
Cons "name" String tail (name :: String, age :: Int, name :: Array String)
```

Mudah-mudahan cukup jelas bagaimana `Cons` ini bekerja karena sebentar lagi kita akan melihat bagaimana kegunaan `Cons` dalam meng-capture informasi type dari suatu record.

## Manipulasi Record Type
### Get
Mari kita latihan otak sekarang! 😄 Kita mulai dengan membuat type signature fungsi yang paling sederhana dulu, `get`, yang kira-kira berfungsi seperti:

```js
const get = (key, record) => record[key]

const result = get('name', { name: 'jihad', age: 26 })
result === 'jihad'
```

Bagaimana mengimplementasikan type signature fungsi tersebut di Purescript dan tetap polymorphic terhadap row? Kita ingin nantinya fungsi `get` di Purescript diakses mirip dengan fungsi Javascript di atas:

```hs
get :: ?belumTau
get = ...

result = get (SProxy :: _ "name") { name: "jihad", age: 26 }
result == "jihad"
```

Ada beberapa yang langkah yang perlu diambil yang menurutku gak susah-susah amat asal udah ngerti konsep `Symbol` dan `Cons`. Langkah pertama adalah dengan mengimplementasikan apa yang bisa dan mudah diimplementasikan, walaupun belum sepenuhnya typechecked:

```hs
get :: ∀ key row a. key -> Record row -> a
get = ...
```

Dimana `key` nantinya akan berupa type level string (Symbol) dan `a` adalah type dari key yang diakses. `row` di sini sudah jelas adalah record itu sendiri.

Langkah kedua adalah memberikan constraint terhadap `key` dengan `SProxy` dan `IsSymbol` persis seperti yang sudah kita bahas di atas 😉

```hs {hl_lines=[2,3]}
get :: ∀ key row a.
  IsSymbol key =>
  SProxy key -> Record row -> a
get = ...
```

We're getting there! Sekarang kita harus membuat koneksi antara `key`, `row`, dan `a` karena sebenarnya mereka adalah satu kesatuan yang tak terpisahkan: **ada sebuah `row` yang memiliki attribute `key` bertipe `a`**. Bagaimana cara mengekspresikan relasi ini?

Dengan `Cons`! Ingat-ingat lagi bahwa `Cons` digunakan untuk mengekspresikan sebuah record yang memiliki _suatu_ attribute beserta type attribute tesebut. Persis seperti yang kita inginkan!

```hs {hl_lines=[3]}
get :: ∀ key row tail a.
  IsSymbol key =>
  Cons key a tail row =>
  SProxy key -> Record row -> a
get = ...
```

Selesai! 🎉 Abaikan saja dulu `tail` di sini dan jangan terlalu dipikirkan, nanti akan ada saatnya kita menggunakan `tail`. Sekarang kita buktikan dulu apakah type signature ini typechecked..

```hs
result :: ?help
result = get (SProxy :: _ "name") { name: "jihad", age: 26 }

-- | Hole 'help' has the inferred type
-- |
-- |     String
```

Nice work, brain 🧠!

### Set
Jom naik level: mari membuat fungsi `set` yang memungkinkan kita untuk mengubah nilai pada suatu attribute sekaligus dapat mengubah type-nya.

```hs
set :: ?belumTau
set = ...

result = set (SProxy :: _ "name") ["jihad", "waspada"] { name: "jihad", age: 26 }
result == { name: ["jihad", "waspada"], age: 26 }
```

Komputasi di atas mengubah type "name" yang semula bertipe `String` menjadi `Array String`. Berarti akan ada duah buah row yang berbeda yang harus dimasukkan ke dalam type signature: row dengan "name" bertipe `String`, dan row dengan "name" bertipe `Array String`. Namun sebelumnya, lakukan upacara dengan Symbol dan kawan-kawannya agar mempermudah langkah selanjutnya.

```hs
set :: ∀ key rowA rowB b.
  IsSymbol key =>
  SProxy key -> b -> Record rowA -> Record rowB
set = ...
```

Lalu asosiasikan `key` dan `b` dengan `rowB` menggunakan teman kita `Cons` karena mereka satu kesatuan republik indonesa:

```hs {hl_lines=[3]}
set :: ∀ key rowA rowB b tail.
  IsSymbol key =>
  Cons key b tail rowB =>
  SProxy key -> b -> Record rowA -> Record rowB
set = ...
```

Lagi, abaikan `tail` untuk saat ini. Sekarang mari kita pikirkan sejenak relasi `rowA` dengan `rowB`. Mereka sebenarnya adalah `row` yang sama, hanya type dari `key`-nya saja yang kemungkinan berbeda. Karena masih ada relasi satu sama lain, kita harus melakukan penggabungan (unifikasi) dua buah insan ini dengan `Cons`:

```hs {hl_lines=[3,4]}
set :: ∀ key rowA a rowB b tail.
  IsSymbol key =>
  Cons key a tail rowA =>
  Cons key b tail rowB =>
  SProxy key -> b -> Record rowA -> Record rowB
set = ...
```

Nah, markicek apa sudah benar implementasi type signature di atas dengan menggunakan fitur type hole.

```hs
result :: ?help
result = set (SProxy :: _ "name") ["jihad", "waspada"] { name: "jihad", age: 26 }

-- |  Hole 'help' has the inferred type
-- |
-- |    { age :: Int
-- |    , name :: Array String
-- |    }
```

Typechecked! ✅

### Delete
Fungsi `delete` menghapus sebuah attribute dari suatu record dan mengembalikan row baru tanpa attribute tersebut. Kita ingin fungsi ini dipanggil seperti:

```hs
delete :: ?belumTau
delete = ...

result = delete (SProxy :: _ "name") { name: "jihad", age: 26 }
result == { age: 26 }
```

Lagi, langkah pertama dalam menulis type signature yang dirasa agak kompleks adalah dengan menuliskan apa yang mudah ditulis.

```hs
delete :: ∀ key rowA rowB.
  IsSymbol key =>
  SProxy key -> Record rowA -> Record rowB
delete = ...
```

`rowA` adalah record yang attribute `key`-nya ingin dihapus, dan `rowB` adalah row baru hasil penghapusan attribute tersebut. Dengan kata lain, `rowB = rowA - key`. PR kita tinggal mengekspresikan relasi ini ke dalam type signature. Dan saya rasa `Cons` masih bisa menjadi jawaban atas problem ini.

Kita review ulang dulu struktur class `Cons` biar freshhhh.

```hs
class Cons (label :: Symbol) (a :: Type) (tail :: # Type) (row :: # Type)

-- `tail` is inferred as (name :: String)
Cons "age" Int tail (name :: String, age :: Int)
^^^^^^^^^^^^^^      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      `head`                  `row`

-- | `head = Cons + label + a`
-- | Sehingga `tail = row - head`
```

Kita dapat melihat pola bahwa **tail adalah row tanpa head**, dimana head sendiri merupakan gabungan dari `Cons`, `label`, dan `a`. Persepsi ini seolah memberikan kesimpulan bahwa `rowB` adalah tail dari `rowA` 😏

```hs {hl_lines=[6]}
-- tail = row - head
-- rowB = rowA - key

delete :: ∀ key a rowA rowB.
  IsSymbol key =>
  Cons key a rowB rowA =>
  SProxy key -> Record rowA -> Record rowB
delete = ...
```

Masih ada relasi yang kelewatan? Kalo gak ada langsung aja kita buktikan apakah type signature di atas typechecked..

```hs
result :: ?help
result = delete (SProxy :: _ "name") { name: "jihad", age: 26 }

-- |  Hole 'help' has the inferred type
-- |
-- |    { age :: Int
-- |    }
```

Typechecked! 🥳

Tapi belum sepenuhnya benar 😄 Ingat bagaimana row bisa menampung label yang duplikat? Yes, kita masih harus benar-benar meyakinkan compiler bahwa `rowB` tidak memiliki label `key`. Caranya dengan memberikan constraint `Lacks`.

Purescript memiliki class bernama `Lacks` yang bisa digunakan untuk mengekspresikan suatu record yang tidak memiliki attribute tertentu.

```hs
class Lacks (label :: Symbol) (row :: # Type)

-- Contoh penggunaan
Lacks "nonExistingKey" (name :: String, age :: Int)
```

Yang bisa dibaca dengan: "Hey compiler, tolong assert bahwa record `(name :: String, age :: Int)` tidak memiliki attribute/label bernama `nonExistingKey`". Karena tidak ditemukan maka expression di atas typechecked. Sebaliknya, compiler akan menolak untuk memberikan lampu hijau jika ditemukan Symbol pada record yang di-assert.

```hs
Lacks "name" (name :: String, age :: Int)

-- | No type class instance was found for
-- |
-- |    Lacks "name"
-- |          ( age :: Int
-- |          , name :: String
-- |          )
```

Oleh karena itu type signature fungsi `delete` masih bisa di-improve lagi dengan memberikan constraint `Lacks`:

```hs {hl_lines=[4]}
delete :: ∀ key a rowA rowB.
  IsSymbol key =>
  Cons key a rowB rowA =>
  Lacks key rowB =>
  SProxy key -> Record rowA -> Record rowB
delete = ...
```

### Insert
Fungsi `insert` juga dapat dibuat dengan mengkombinasikan class `Lacks` dengan `Cons`. Intuisi type signature fungsi `insert` ini saya serahkan ke pembaca untuk exercise 🙂

(( sebenernya mager sih jelasin panjang lebar lagi, takut kepanjangan artikelnya hehe ))

```hs
insert :: ∀ key a rowA rowB.
  IsSymbol key =>
  Lacks key rowA =>
  Cons key a rowA rowB =>
  SProxy key -> a -> Record rowA -> Record rowB
insert = ...

typeChecked :: { name :: String, age :: Int, isGanteng :: Boolean }
typeChecked = insert (Symbol :: _ "isGanteng") true { name: "Jihad", age: 26 }

error = insert (Symbol :: _ "name") "Waspada" { name: "Jihad", age: 26 }
```

## Wrap Up
Purescript secara default menyediakan beberapa class dan data structure yang dapat digunakan untuk memanipulasi informasi rows di type level. Umumnya mereka bisa ditemukan di module `Prim.Row` dan `Prim.RowList`.

Kalo temen-temen mau lihat fungsi-fungsi lain untuk record manipulation sebagai inspirasi, mungkin bisa mampir ke [purescript-record](https://github.com/purescript/purescript-record). Dokumentasi dan Doc Comment-nya cukup jelas dan mudah diikuti.

Saya harap pembahasan di artikel ini gampang dicerna dan gak terlalu kompleks. Dan yang paling penting, semoga masih bisa memberi bermanfaat. Sayonara ✌🏻
