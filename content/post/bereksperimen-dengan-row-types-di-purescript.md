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

Belakangan saya lagi iseng-iseng main metaprogramming sama record system-nya Purescript, gimana cara nambahin value, ngubah type suatu attribute (key), menghilangkan attribute, dan lain-lain. Basic idea-nya sama kayak artikel saya yang lalu soal Row Polymorphism di Typescript. Tapi kali ini pake Purescript: gimana caranya operasi-operasi tersebut dapat ditangkap dan di-infer oleh compiler tanpa kehilangan type-safety.

Artikel ini gak akan berfokus pada akses record di Purescript, tapi lebih ke arah type-level programming untuk manipulasi record. Jadi contoh-contoh di bawah nanti gak akan saya sertakan implementation details-nya, hanya type signature-nya saja.

**NOTE**: Kalau temen-temen menemukan kata "attribute", "label", atau "key" di artikel ini, perlu saya garisbawahi bahwa saya merujuk pada hal yang sama :))

## Symbol
Sebelum masuk ke pembahasan yang lebih lanjut, kita harus mengetahui terlebih dahulu apa maksud Symbol di Purescript. Symbol yang ini jangan disamakan dengan Symbol yang ada di Javascript. Symbol di Puresctipt digunakan untuk merepresentasikan type-level string.

Nah `Proxy` dapat membantu kita membuat type-level string dan function `reflectSymbol` untuk "menurunkan derajat" dari type-level string kembali ke term-level string.

```purs
termLevel = "Jihad"

typeLevel :: Proxy "Jihad"
typeLevel = Proxy

jihad = reflectSymbol typeLevel
jihad == termLevel
```

{{< figure src="https://media.giphy.com/media/12NUbkX6p4xOO4/giphy.gif" alt="one on one" caption="Compiler magic" class="fig-center img-60" >}}

Oh ya satu lagi, `reflectSymbol` merupakan method dari class `IsSymbol` sehingga harus digunakan sebagai constraint bila kita berniat memanggil `reflectSymbol`.

Pada section berikutnya kita akan melihat kombinasi antara `IsSymbol` dan `Proxy` banyak digunakan untuk merujuk pada suatu attribute record di level type.

## Cons
Sewaktu artikel ini ~~ditulis (v13.3)~~ di-update (v14.4), Purescript memiliki beberapa utility class untuk dapat memanipulasi dan menangkap informasi record saat compile-time. Salah satu yang paling sering digunakan adalah `Cons`.

```purs
-- module Prim.Row

class Cons :: Symbol -> Type -> Row Type -> Row Type
class Cons label a tail row
  | label a tail -> row
  , label row -> a tail
```

class `Cons` mengekspresikan **sebuah record yang memiliki attribute `label` yang bertipe `a`**. `tail` adalah sisa `row` tanpa `label`.

Perlu diperhatikan bahwa row bisa memiliki label yang duplikat di type level. Saya udah nanya soal kenapa row types bisa mengandung duplikasi label di slack channel tapi pak Harry sendiri juga gatau kenapa haha. Yo-uwes, mungkin kapan-kapan dijawab sama yang lebih berwenang.

Anyway, `Cons` ini bisa diibaratkan seperti ini:

```purs
class Cons :: Symbol -> Type -> Row Type -> Row Type
class Cons label a tail row
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
Mari kita latihan otak sekarang! Kita mulai dengan membuat type signature fungsi yang paling sederhana dulu, `get`, yang kira-kira berfungsi seperti:

```js
const get = (key, record) => record[key]

const result = get('name', { name: 'jihad', age: 26 })
result === 'jihad'
```

Bagaimana mengimplementasikan type signature fungsi tersebut di Purescript dan tetap row-polymorphic? Kita ingin nantinya fungsi `get` di Purescript diakses mirip dengan fungsi Javascript di atas:

```purs
get :: ?belumTau
get = ...

result = get (Proxy :: _ "name") { name: "jihad", age: 26 }
result == "jihad"
```

Ada beberapa yang langkah yang perlu diambil yang menurutku gak susah-susah amat asal udah ngerti konsep `Symbol` dan `Cons`. Langkah pertama adalah dengan mengimplementasikan apa yang bisa dan mudah diimplementasikan, walaupun belum sepenuhnya typecheck:

```purs
get :: ∀ key row a. key -> Record row -> a
get = ...
```

Dimana `key` nantinya akan berupa type level string (Symbol) dan `a` adalah type dari key yang diakses.

Langkah kedua, adalah dengan mengubah `key` menjadi `Proxy key` karena kita ingin string key yang di-supply dikenali oleh type checker.

```purs {hl_lines=[2]}
get :: ∀ key row a.
  Proxy key -> Record row -> a
get = ...
```

We're getting there! Sekarang kita harus membuat koneksi antara `key`, `row`, dan `a` karena sebenarnya mereka adalah satu kesatuan yang tak terpisahkan: **ada sebuah `row` dengan attribute `key` yang bertipe `a`**. Bagaimana cara mengekspresikan relasi ini?

Dengan `Cons`! Ingat bahwa `Cons` digunakan untuk mengekspresikan sebuah record yang memiliki _suatu_ attribute tertentu.

```purs {hl_lines=[2]}
get :: ∀ key row tail a.
  Cons key a tail row =>
  Proxy key -> Record row -> a
get = ...
```

Dan terakhir, dengan asumsi kita memiliki sebuah FFI sebagai implementation details untuk dieksekusi saat runtime, kita harus menambahkan constraint `IsSymbol` agar type-level key di parameter pertama dapat direalisasikan sebagai string biasa.

```purs {hl_lines=[4]}
foreign import getImpl :: forall row a. String -> Record row -> a

get :: ∀ key row tail a.
  IsSymbol key =>
  Cons key a tail row =>
  Proxy key -> Record row -> a
get key r = getImpl (reflectSymbol key) r
```

dimana `getImpl` sendiri adalah

```js
exports.getImpl = key => record => record[key]
```

Selesai! 🎉 Abaikan saja dulu `tail` di sini dan jangan terlalu dipikirkan, nanti akan ada saatnya kita menggunakan `tail`. Sekarang kita lihat dulu apakah compiler dapat meng-infer type yang tepat dengan type signature ini..

```purs
result :: ?help
result = get (Proxy :: _ "name") { name: "jihad", age: 26 }

-- | Hole 'help' has the inferred type
-- |
-- |     String
```

Nice work, brain 🧠!

### Set
Jom naik level: mari membuat fungsi `set` yang memungkinkan kita untuk mengubah nilai pada suatu attribute sekaligus dapat mengubah type-nya.

```purs
set :: ?belumTau
set = ...

result :: { name :: Array String, age :: Int }
result = set name ["jihad", "waspada"] { name: "jihad", age: 26 }
  where name = (Proxy :: _ "name")

result == { name: ["jihad", "waspada"], age: 26 }
```

Komputasi di atas mengubah type "name" yang semula bertipe `String` menjadi `Array String`. Berarti akan ada duah buah row yang berbeda yang harus dimasukkan ke dalam type signature: row dengan "name" bertipe `String`, dan row dengan "name" bertipe `Array String`. Namun sebelumnya, lakukan upacara dengan Symbol dan kawan-kawannya agar mempermudah langkah selanjutnya.

```purs
set :: ∀ key rowA rowB b.
  IsSymbol key =>
  Proxy key -> b -> Record rowA -> Record rowB
set = ...
```

Lalu asosiasikan `key` dan `b` dengan `rowB` menggunakan teman kita `Cons` karena mereka satu kesatuan republik indonesa:

```purs {hl_lines=[3]}
set :: ∀ key rowA rowB b tail.
  IsSymbol key =>
  Cons key b tail rowB =>
  Proxy key -> b -> Record rowA -> Record rowB
set = ...
```

Lagi, abaikan `tail` untuk saat ini. Sekarang mari kita pikirkan sejenak relasi `rowA` dengan `rowB`. Mereka sebenarnya adalah `row` yang sama, hanya type dari `key`-nya saja yang kemungkinan berbeda. Karena masih ada relasi satu sama lain, kita harus melakukan penggabungan dua buah record ini dengan `Cons`:

```purs {hl_lines=[3,4]}
set :: ∀ key rowA a rowB b tail.
  IsSymbol key =>
  Cons key a tail rowA =>
  Cons key b tail rowB =>
  Proxy key -> b -> Record rowA -> Record rowB
set = ...
```

Dengan penambahan constraint ini, typechecker dapat melihat relasi antara keduanya dengan benar: bahwa keduanya memiliki `key` dan `tail` yang sama, hanya type dari `key`-nya saja yang berbeda.

Nah, markicek apa sudah benar implementasi type signature di atas dengan menggunakan fitur type hole.

```purs
result :: ?help
result = set name ["jihad", "waspada"] { name: "jihad", age: 26 }
  where name = (Proxy :: _ "name")

-- |  Hole 'help' has the inferred type
-- |
-- |    { age :: Int
-- |    , name :: Array String
-- |    }
```

Typechecked! ✅

### Delete
Fungsi `delete` menghapus sebuah attribute dari suatu record dan mengembalikan row baru tanpa attribute tersebut. Kita ingin fungsi ini dipanggil seperti:

```purs
delete :: ?belumTau
delete = ...

result = delete (Proxy :: _ "name") { name: "jihad", age: 26 }
result == { age: 26 }
```

Lagi, langkah pertama dalam menulis type signature yang dirasa agak kompleks adalah dengan menuliskan apa yang mudah ditulis.

```purs
delete :: ∀ key rowA rowB.
  IsSymbol key =>
  Proxy key -> Record rowA -> Record rowB
delete = ...
```

`rowA` adalah record yang attribute `key`-nya ingin dihapus, menghasilkan `rowB`. Dengan kata lain, `rowB = rowA - key`. PR kita tinggal mengekspresikan relasi ini ke dalam type signature. Dan saya rasa `Cons` masih bisa menjadi jawaban atas problem ini.

Kita review ulang dulu struktur class `Cons` biar freshhhh.

```purs
Cons "age" Int tail (name :: String, age :: Int)
     ^^^^^^^^^      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      `head`                  `row`

-- `tail` is inferred as (name :: String)
```

Kita dapat melihat pola bahwa **tail adalah row tanpa head**. Persepsi ini seolah memberikan kesimpulan bahwa `rowB` adalah tail dari `rowA`.

```purs {hl_lines=[6]}
-- tail = row - head
-- rowB = rowA - key

delete :: ∀ key a rowA rowB.
  IsSymbol key =>
  Cons key a rowB rowA =>
  Proxy key -> Record rowA -> Record rowB
delete = ...
```

Masih ada relasi yang kelewatan? Kalo gak ada langsung aja kita buktikan apakah type signature di atas typechecked..

```purs
result :: ?help
result = delete (Proxy :: _ "name") { name: "jihad", age: 26 }

-- |  Hole 'help' has the inferred type
-- |
-- |    { age :: Int
-- |    }
```

Typechecked!

Tapi belum sepenuhnya benar. Ingat bagaimana row bisa menampung label yang duplikat? Yes, kita masih harus benar-benar meyakinkan compiler bahwa `rowB` tidak memiliki label `key`. Caranya dengan memberikan constraint `Lacks`.

Purescript memiliki class bernama `Lacks` yang bisa digunakan untuk mengekspresikan suatu record yang tidak memiliki attribute tertentu.

```purs
class Lacks (label :: Symbol) (row :: Row Type)

-- Contoh penggunaan
Lacks "nonExistingKey" (name :: String, age :: Int)
```

Yang bisa dibaca dengan: "Hey compiler, tolong assert bahwa record `(name :: String, age :: Int)` tidak memiliki attribute/label bernama `nonExistingKey`". Karena tidak ditemukan maka expression di atas typechecked. Sebaliknya, compiler akan menolak untuk memberikan lampu hijau jika ditemukan Symbol pada record yang di-assert.

```purs
Lacks "name" (name :: String, age :: Int)

-- | No type class instance was found for
-- |
-- |    Lacks "name"
-- |          ( age :: Int
-- |          , name :: String
-- |          )
```

Oleh karena itu type signature fungsi `delete` masih bisa di-improve lagi dengan memberikan constraint `Lacks`:

```purs {hl_lines=[4]}
delete :: ∀ key a rowA rowB.
  IsSymbol key =>
  Cons key a rowB rowA =>
  Lacks key rowB =>
  Proxy key -> Record rowA -> Record rowB
delete = ...
```

### Insert
Fungsi `insert` juga dapat dibuat dengan mengkombinasikan class `Lacks` dengan `Cons`. Intuisi type signature fungsi `insert` ini saya serahkan ke pembaca untuk exercise 🙂

```purs
insert :: ∀ key a rowA rowB.
  IsSymbol key =>
  Lacks key rowA =>
  Cons key a rowA rowB =>
  Proxy key -> a -> Record rowA -> Record rowB
insert = ...

typeChecked :: { name :: String, age :: Int, isGanteng :: Boolean }
typeChecked = insert isGanteng true { name: "Jihad", age: 26 }
  where isGanteng = (Symbol :: _ "isGanteng")

error = insert name "Waspada" { name: "Jihad", age: 26 }
  name = (Symbol :: _ "name")
```

## Wrap Up
Purescript secara default menyediakan beberapa class dan data structure yang dapat digunakan untuk memanipulasi informasi rows di type level. Umumnya mereka bisa ditemukan di module `Prim.Row` dan `Prim.RowList`.

Kalo temen-temen mau lihat fungsi-fungsi lain untuk record manipulation sebagai inspirasi, mungkin bisa mampir ke [purescript-record](https://github.com/purescript/purescript-record). Dokumentasi dan Doc Comment-nya cukup jelas dan mudah diikuti.

Saya harap pembahasan di artikel ini gampang dicerna dan gak terlalu kompleks. Dan yang paling penting, semoga masih bisa memberi bermanfaat. Sayonara ✌🏻
