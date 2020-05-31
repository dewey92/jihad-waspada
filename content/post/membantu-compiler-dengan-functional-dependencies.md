---
title: "Membantu Compiler dengan Functional Dependencies"
date: 2019-08-16T13:15:43+02:00
description: "Functional Dependencies memungkinkan programmer mengekspresikan relasi antar type sekaligus memberi compiler jalan pintas dalam meng-infer suatu type"
images: ["/uploads/rope-climbing.jpg"]
image:
  src: "/uploads/rope-climbing.jpg"
  caption: Image by <a href="https://pixabay.com/users/skeeze-272447/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1761386">skeeze</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1761386">Pixabay</a>
tags: ["purescript", "haskell", "types"]
categories: ["programming", "type system", "purescript"]
draft: false
---

Udah cukup banyak artikel yang menjelaskan tentang apa itu Functional Dependencies &mdash; salah satu fitur type system di Haskell dan Purescript &mdash; beserta use cases-nya. Namun yang tak rasakan justru kurang mudah dipahami dan sulit dianalogikan dengan studi kasus lain. Liat aja contoh dari [Haskell Wiki](https://wiki.haskell.org/Functional_dependencies) sendiri, FuncDep dijelaskan dengan studi kasus Vector dan Matriks yang Vector sendiri aja gue gak paham 🤣 Nah, artikel ini murni tak coba tulis untuk mematangkan pemahaman saya pribadi sekaligus berharap ada masukan dan kritik dari teman-teman yang lebih paham.

---
## Multi-parameter Type Classes

Lahirnya fitur Functional Dependencies ini katanya terinsipirasi dari [fitur Functional Dependencies yang ada pada relational database](https://opentextbc.ca/dbdesign01/chapter/chapter-11-functional-dependencies/), yaitu relasi attribute antar table (biasanya berurusan dengan Primary Key). Ada One to One, One to Many, Many to One, dan Many to Many. Relasi dalam database ini dan korelasinya dengan FuncDep akan di bahas di bawah. Sekarang mari bahas Multi-param type classes dulu.

Functional Dependencies baru dapat digunakan ketika kita menulis class yang memiliki type parameter lebih dari satu, alias _multi-parameter type class_.

```hs
class MultiParamTypeClass a b

-- `a` dan `b` merupakan type parameter dari class `MultiTypeParamClass`
```

"Baru dapat digunakan" dalam arti kita bisa memilih untuk menggunakan fitur FuncDep atau tidak, tergantung kasus-nya. Jika tidak menggunakan FuncDep, maka `a` dan `b` dapat diisi dengan type apa saja tanpa ada restriction, seperti relasi many-to-many pada database. Any combination of `a` and `b` is valid. Akan berbeda ketika FuncDep digunakan:

```hs
class MultiParamTypeClass a b | a -> b
                              ^^^^^^^^
```

Di sini compiler bilang: "gue bakal bisa infer type `b` asal gue tau dulu apa type `a`". Dengan kata lain `a` uniquely determines `b`.

Setidaknya itu yang banyak tak baca di internet, yang awalnya justru membuat saya merasa bersalah karena tau artinya secara literal tapi tidak secara kontekstual.

## Many to One
Nah seperti yang sudah dijelaskan di atas (bahwa FuncDep terinspirasi dari database), maksud dari notasi `a -> b` dapat dipahami menggunakan intuisi yang sama ketika memahami relasi many-to-one pada relational database.

Misal ketika bicara Geografi, many-to-one bisa dianalogikan dengan relasi kota terhadap provinsi: satu kota hanya ada pada satu provinsi, tapi satu provinsi dapat memiliki banyak kota.

```hs
class ManyToOne city province | city -> province
```

Kalau saya tanya ke kamu: "Surabaya ada di Provinsi apa?" Pasti jawabannya cuman satu dan memang satu-satunya jawaban, Jawa Timur. Tapi sebaliknya kalau ditanya: "Jawa Timur kotanya apa?", kita gak akan bisa jawab hanya dengan satu kota saja karena jawabannya banyak, akan ada Surabaya, Malang, Jember, Banyuwangi, dan kota-kota lainnya.

Maka betul kata compiler tadi: "gue bakal bisa infer type `province` asal gue tau dulu apa type `city`". Tapi _darimana_ compiler bisa tau? 🤔

Dari instance yang kita tulis 👇🏻

```hs
-- class with funcdep
class ManyToOne city province | city -> province

-- instance
instance surabaya :: ManyToOne Surabaya JawaTimur
instance malang :: ManyToOne Malang JawaTimur
instance jember :: ManyToOne Jember JawaTimur
instance semarang :: ManyToOne Semarang JawaTengah
```

Jadi sebenarnya tidak ada magic di sini: kita yang mendikte compiler.

Bagaimana kalau saya, misal, mau bilang Surabaya itu **juga** bagian dari Jawa Tengah? Mungkin pipi saya bakal jadi merah digampar sama Pak Guru. Alasannya? IQ lu jongkok! 🙊

```hs
instance sbyJatim :: ManyToOne Surabaya JawaTimur
instance sbyJateng :: ManyToOne Surabaya JawaTengah
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
{--
Overlapping type class instances found for

    ManyToOne Surabaya JawaTengah

  The following instances were found:

    sbyJatim
    sbyJateng


in type class instance

  ManyToOne Surabaya JawaTengah
--}
```

Dari sini bisa ditarik kesimpulan, bahwa functional dependencies berguna untuk membatasi jumlah instance type variable yang muncul **di sebelah kiri** arrow (`a` pada `a -> b` atau `city` pada `city -> province`). Dengan kata lain type variable di sebelah kiri arrow **harus bersifat unique**, tidak boleh muncul di class instance lebih dari satu kali. Tanpa FuncDep, code di atas akan dianggap valid oleh compiler.

Kasus yang paling umum di dunia nyata adalah ketika menggunakan MTL `MonadThrow` yang menspesifikasikan bahwa setiap Monad yang implements class `MonadThrow` harus memiliki satu error type saja, tidak boleh lebih.

```hs
class Monad m <= MonadThrow e m | m -> e where
  throwError :: ∀ a. e -> m a

-- Monad `Effect` harus memiliki satu error type saja: `Error`
instance monadThrowEffect :: MonadThrow Error Effect where
  throwError = ...

-- Monad `Aff` harus memiliki satu error type saja: `Error`
instance monadThrowAff ∷ MonadThrow Error Aff where
  throwError = ...

-- Custom monad seperti `TestM`, bisa menggunakan String sebagai error type-nya
instance monadThrowTestM ∷ MonadThrow String TestM where
  throwError = ...
```

Dengan begini, compiler juga dapat langsung mengetahui apa type `e` begitu type `m` diketahui. Artinya, jika nanti ada function yang menggunakan `Aff` monad dan memanggil `throwError` di dalamnya, compiler bisa langsung tahu bahwa `e` pastilah bertipe `Error` dan bukan yang lain. Demikian pula ketika ada function di dalam konteks `TestM`, begitu ada pemanggilan `throwError` compiler akan langsug bisa meng-infer `e` sebagai `String`.

## One to One
Fitur Functional Dependencies juga bisa memiliki spesifikasi yang lebih narrow dari relasi many-to-one, yaitu one-to-one.

{{< figure src="https://media.giphy.com/media/HIhwVDyNC6JOw/giphy-downsized.gif" alt="one on one" caption="One on One ini mah bro" class="fig-center img-60" >}}

Dengan one-to-one, kita menjamin jumlah instance type variable `a` **dan** `b` hanya satu saja. Mereka tidak boleh muncul lebih dari satu kali. Masih berkaitan dengan geografi, contoh yang paling mudah adalah relasi antara ibu kota dengan negaranya: suatu negara hanya boleh memiliki satu ibu kota **dan** suatu ibu kota hanya boleh dimiliki oleh satu negara.

```hs
class OneToOne capital country | capital -> country, country -> capital

instance indo :: OneToOne Jakarta Indonesia
instance murica :: OneToOne Washington USA
instance londo :: OneToOne Amsterdam Belanda

instance jakMalay :: OneToOne Jakarta Malaysia -- error, Jakarta punya Indo, jangan maling!
instance berlinIndo :: OneToOne NewYork Indonesia -- error, Indo sudah punya Jakarta
```

Ketika ada pertanyaan: "Apa ibu kota negara +62?". Jawabannya pasti hanya satu yaitu Jakarta. Dan ketika ditanya balik: "Jakarta ibu kota negara apa?". Jawabannya juga hanya satu yaitu negara +62.

Dengan relasi seperti ini, compiler bisa menginfer salah satu type asal type yang satunya sudah diketahui. Relasi one-to-one ini juga bisa disebut Bidirectional Dependencies.

## Extra Type Variable
Section ini masih berkaitan dengan many-to-one relationship, yang membedakan hanyalah jumlah type parameter di sebelah kiri atau kanan arrow. Kalau sebelumnya hanya satu (seperti `a -> b`), yang ini bisa lebih dari satu (seperti `a b -> c`). Namun tetap tidak mengubah arti: jika compiler tahu apa type `a` dan `b`, maka type `c` otomatis dapat langsung diketahui.

Anggap kita ingin mengimplementasikan behaviour `+` di Javascript yang dapat menerima lebih dari satu type: Float (`Number` di Purescript), dan String. Yang secara umum dapat direpresentasikan sebagai berikut:

| +          | Number | String |
|------------|--------|--------|
| **Number** | Number | String |
| **String** | String | String |

Dengan aturan ini, kita dapat melihat setidaknya ada dua buah pola menarik:

1. Kombinasi type kedua buah operand (in bold) bersifat unique. Tidak ada **kombinasi** yang muncul lebih dari sekali
2. Type hasil penjumlahan dengan operator `+` ditentukan oleh kedua buah type operand

Code-wise, aturan tersebut dapat dituliskan dengan sebuah class yang menerima tiga buah type variable.

```hs
class JavascriptPlus a b c | a b -> c where
  jplus :: a -> b -> c

-- Unique instances
instance jNumNum :: JavascriptPlus Number Number Number where
  jplus = -- whatever
instance jNumStr :: JavascriptPlus Number String String where
  jplus = -- whatever
instance jStrNum :: JavascriptPlus String Number String where
  jplus = -- whatever
instance jStrStr :: JavascriptPlus String String String where
  jplus = -- whatever
```

Sekarang malah keliatan kayak pattern-matching tapi di level type 😅 "Kalo aku punya String dan Number, maka hasilnya harus String" dan seterusnya. Tapi yang pasti, function `jplus` ini mengembalikan return type yang berbeda-beda tergantung "input"-nya.

```hs
toUpper :: String -> String

resInNum = 5.0 `jplus` 6.0 -- 11.0 :: Number
resInStr = 5.0 `jplus` "6" -- "56" :: String

typeChecked = toUpper <<< resInStr -- passed ✅
notCompiled = toUpper <<< resInNum -- error! ❌
```

Kasus Functional Dependencies dengan type parameter lebih dari dua ini bisa ditemukan ketika bermain dengan Record. Seperti type [Union](https://pursuit.purescript.org/builtins/docs/Prim.Row#t:Union) yang memiliki type signature:

```hs
class Union (left :: # Type) (right :: # Type) (union :: # Type)
  | left right -> union
  , right union -> left
  , union left -> right
```

Yang pada dasarnya hanya melakukan penggabungan 2 buah row dan menghasilkan sebuah row baru hasil penggabungan tersebut. Intuisi selanjutnya di balik type Union ini saya kembalikan ke masing-masing pembaca.

## Function Overloading
Dari contoh di atas FuncDep sekilas terlihat seperti function overloading! Dan memang benar, FuncDep "bisa" digunakan untuk meng-encode function overloading seperti yang lumrah ada pada bahasa pemrograman lain. Berikut perbandingan yang identik di Typescript:

```ts
declare function jplus(x: number, y: number): number;
declare function jplus(x: number, y: string): string;
declare function jplus(x: string, y: number): string;
declare function jplus(x: string, y: string): string;

const intint = jplus(5, 6);     // inferred as `number`
const intstr = jplus(5, '6');   // inferred as `string`
const strint = jplus('5', 6);   // inferred as `string`
const strstr = jplus('5', '6'); // inferred as `string`
```

Yang membedakan, function overloading ini tidak memiliki _relasi_ antar type seperti yang ada pada FuncDep. Program di bawah ini typecheck, walaupun jika dijalankan, overload yang terakhir tidak akan pernah dipanggil.

```ts
declare function jplus(x: string, y: string): string;
declare function jplus(x: string, y: string): number; // never been called

const strstr = jplus('5', '6'); // inferred as `string`
```

Sedangkan compiler Purescript sendiri akan menolak fungsi di atas (jika menggunakan FuncDep) karena kombinasi type `x` dan `y` overlap (tidak unique).

---

Namun Functional Depndencies lebih dari sekedar function overloading. Contoh yang real worldish adalah ketika mencoba re-implement State Monad dan membuat instance dengan Ref (mutable variables).

```hs
import Effect.Ref as Ref

class SM m r | m -> r, r -> m where
  new :: ∀ a. a -> m (r a)
  read :: ∀ a. r a -> m a
  write :: ∀ a. a -> r a -> m Unit

instance smEffect :: SM Effect Ref.Ref where
  new = Ref.new
  read = Ref.read
  write = Ref.write
```

Bagian `m -> r, r -> m` mengindikasikan adanya dua buah FuncDeps (Bidirectional Dependencies) sekaligus mengekspresikan relasi one-to-one. Sekarang perhatikan code berikut:

```hs
someFn x = do
  r <- new x  -- `SM` monad
  log "Hello" -- `Effect` monad
  pure r
```

Adanya pemanggilan fungsi `log` (yang memiliki type `String -> Effect Unit`) berimplikasi pada asumsi bahwa fungsi `someFn` ada di dalam `Effect` monad, yang, kalau dilihat dari Functional Dependencies-nya, type `r` pasti merujuk pada `Ref.Ref`. Dan pada akhirnya, compiler akan dengan sendirinya meng-infer fungsi `someFn` sebagai

```hs
someFn :: ∀ a. a -> Effect (Ref.Ref a)
someFn x = ...
```

Andaikan FuncDep tidak digunakan, fungsi `someFn` akan memiliki type

```hs
someFn :: ∀ r a. SM Effect r => a -> Effect (r a)
someFn x = ...
```

yang tentunya tidak diinginkan karena terlalu general. Di lain kasus, tidak adanya FuncDep dapat menimbulkan ambiguity di sisi compiler.

```hs
ambiguousFn :: ∀ a. a -> Effect a
ambiguousFn x = do
  r <- new x
  read r

-- Compiler akan complain
{--
No type class instance was found for

  SM Effect t5

The instance head contains unknown type variables.
Consider adding a type annotation.
--}
```

## Kesimpulan
Functional Dependencies bisa digunakan oleh programmer ketika ingin memberikan constraint terhadap type saat proses type inference dengan mendeklarasikan relasi di multi-param type class (one-to-one, many-to-one). Dengan FuncDep compiler dapat didikte/dibantu untuk mengetahui type mana yang **bisa langsung** di-infer dari type lain. Yaa itung-itung amal baik ke compiler yang selama ini sudah banyak ngebantu report error sana sini 😁

Mudah-mudahan artikel ini dapat membantu memahami motivasi dan kegunaan dari Functional Dependencies. Ciao 👋🏻
