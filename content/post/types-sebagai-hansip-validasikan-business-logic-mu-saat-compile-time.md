---
title: "Types sebagai Hansip: Validasikan Business Logic-mu saat Compile Time"
date: 2019-08-30T11:18:00+02:00
description: "Berbagi beban dengan compiler untuk memastikan business requirement terimplementasikan dengan benar"
images: ["/uploads/hansip.jpg"]
image:
  src: "/uploads/hansip.jpg"
  caption: Image by <a href="https://pixabay.com/users/RyanMcGuire-123690/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=869216">Ryan McGuire</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=869216">Pixabay</a>
tags: ["purescript", "types"]
categories: ["programming", "purescript", "type system"]
draft: false
---

Sekitar setahun yang lalu dari penulisan artikel ini, saya pernah menulis artikel tentang [Nominal Typing (di Typescript)](https://medium.com/codewey/typescript-nominal-typing-99c1d91a891c) yang berguna untuk membedakan satu type dengan type lainnya walaupun struktur data keduanya identik. Dari sudut pandang lain, artikel tersebut juga meyebutkan bahwa Nominal Typing dapat digunakan untuk membatasi programmer dari kemungkinan melakukan kesalahan-kesalahan business domain sekaligus meningkatkan code expressiveness dengan memanfaatkan type system.

Artikel ini kurang lebih membahas konsep yang sama namun ditelaah menggunakan type system di Purescript.

## Masalah dengan String dan Angka
Anggap ada requirement project yang melibatkan konsep mata uang. Katakanlah Rupiah dan Euro. Dan ada function `addMoney` yang melakukan penjumlahan dua buah nominal uang.

```hs
addMoney :: Number -> Number -> Number
addMoney a b = a + b
```

Pemodelan fungsi seperti ini terlalu loose dan berbahaya karena tidak mampu mencegah programmer ketika melakukan penjumlahan dua buah mata uang yang berbeda. Kita gak mau kan secara naif menjumlahkan Rupiah dengan Euro karena Rp1 jelas gak sama dengan €1. Hal ini dapat menyebabkan komputasi yang tidak akurat dan bisa fatal kalau teledor 🥶

Salah satu cara mungkin bisa dengan menamakan kedua buah variable dengan penamaan yang jelas.

```hs
oneRupiah = 1.0
oneEuro = 1.0
```

Ini pun masih error-prone dan tidak menyentuh akar masalah. Fungsi `addMoney` masih bisa dipanggil dan gak bakal ngasih warning apa-apa ke programmer. Dengan kata lain kita **masih memberikan ruang bagi data yang tidak valid** untuk diproses. Big NO.

Hal yang terlihat sepele ini mengingatkan saya akan insiden tahun 1998 dimana NASA (_hello flat-earthers!_) merugi besar ketika kehilangan Martian Rover dalam ekspedisinya ke Mars [hanya karena perbedaan metrics unit dalam kalkulasinya](https://www.vice.com/en_us/article/wnjbqb/the-story-of-beagle-2-the-long-lost-martian-rover-thats-finally-been-found). Duh pak 😓

Pola yang sama juga bisa terjadi pada string. String has too much power. Ia mampu merepresentasikan apa saja yang bisa dibayangkan: karakter, kata, kalimat, nama, email, alamat, number (!!!), function, you name it.

Eh, function? Iya, function.

```js
// javascript
eval("function fnName() { return 'jihad' }")

fnName() === 'jihad'
```

👻👻👻

_Being powerful is great!_ Tapi perlu dicatat bahwa power yang sesungguhnya itu bukan yang digunakan tanpa batas, the real power comes when you can control it 😎 Iyadah serah lu!

{{% figure src="https://media.giphy.com/media/42wMv7pMjAnH4uBn8G/giphy-downsized.gif" alt="yeah right" caption=" " class="fig-center img-60" %}}

> Bukanlah orang kuat yang sebenarnya dengan (selalu mengalahkan lawannya dalam) perkelahian, tetapi tidak lain orang **kuat** yang sebenarnya adalah yang mampu **mengendalikan** dirinya ketika marah. - Muhammad SAW

Udah dong ceramahnya pak haji, balik ke programming.

Sama kayak programming, menurut saya sesuatu bisa dibilang **powerful** ketika dapat menyediakan **constraint** untuk membatasi programmer dari kemungkinan melakukan hal-hal gila haha. Dalam hal ini, constraint tersebut adalah type system.

## Newtype
Di Purescript, `newtype` bersifat opaque yang berarti walaupun secara struktur dan runtime representation-nya persis sama namun mereka dianggap berbeda saat compile time.

{{< highlight hs "hl_lines=4-5 10-11" >}}
-- fileA.purs
module FileA where

newtype Rupiah = Rupiah Number
derive newtype instance eqRupiah :: Eq Rupiah

-- fileB.purs
module FileB where

newtype Rupiah = Rupiah Number
derive newtype instance eqRupiah :: Eq Rupiah

-- main.purs
import FileA as FileA
import FileB as FileB

serebu = FileA.Rupiah 1000.0
seceng = FileB.Rupiah 1000.0

result = serebu != seceng -- error ❌

-- | Could not match type
-- |
-- |   Rupiah
-- |
-- | with type
-- |
-- |   Rupiah
{{< /highlight >}}

Newtype sangat berguna untuk membuat distinction dari satu tipe data yang sama. Seperti ketika ingin membedakan konsep `name`, `address`, `url` yang biasanya diekspresikan dengan string biasa.

```hs
newtype Name = Name String
newtype Address = Address String
newtype URL = URL String

yell :: Name -> String
yell (Name x) = toUpper x

valid = yell (Name "jihad") == "JIHAD" -- typechecked ✅

invalid  = yell (Address "Tangerang")   -- error ❌
invalid' = yell (URL "https://url.com") -- error ❌
```

Kembali ke masalah utama. Dengan teknik ini, kita bisa dengan mudah memebedakan mata uang Rupiah dengan Euro!

```hs
newtype Rupiah = Rupiah Number
newtype Euro = Euro Number
```

Cuman masalahnya, bagaimana type signature dari fungsi `addMoney`???

```hs
addMoney :: Rupiah -> Rupiah -> Rupiah
addMoeny :: Rupiah -> Euro -> Rupiah
...
addMoney :: Euro -> Euro -> Euro
```

Nope. Bukan ini yang kita mau. Kita ingin type signature fungsi `addMoney` polymorphic terhadap berbagai jenis mata uang.

## Kind Signature
Untuk membuat fungsi `addMoney` bekerja dengan `Rupiah` dan `Euro`, mereka harus dikelompokkan ke dalam sesuatu yang sama. Mari kita sebut `Currency`.

```hs
newtype Currency a = Amount Number

addMoney :: ∀ a. Currency a -> Currency a -> Currency a
addMoney (Amount x) (Amount y) = Amount (x + y)
```

Kita bisa lihat bahwa type variable `a` tidak muncul di sebalah kanan persamaan, yang membuat type `Currency` ini **Phantom Type**. Lalu gunanya `a` ini apa?

Type variable `a` berguna sebagai tag, sebagai pembeda antara satu mata uang dengan mata uang lainnya.

```hs
tenEuro :: Currency "euro"
tenEuro = Amount 10.0

tenRupiah :: Currency "rupiah"
tenRupiah = Amount 10.0
```

> Saya sarankan pembaca yang belum familiar dengan kind untuk memahami konsep [kind di Purescript]({{< ref "/post/term-type-dan-kind-di-purescript" >}}).

Sekarang type variable `a` sudah diisi oleh type level string (Symbol): "euro" dan "rupiah". Kita perlu modifikasi definisi Currency sedikit karena, seperti yang kita tahu, type variable di Purescript by default memiliki kind `Type` sedangkan type level string punya kind `Symbol`.

```hs
newtype Currency (a :: Symbol) = Amount Number
```

Lalu batasi export data constructor Currency dan buat factory function sesuai kebutuhan project.

```hs
module Project.Currency
  ( Currency -- data constructor tidak di-export
  , rupiah   -- factory function untuk Rupiah
  , euro     -- factory function untuk Euro
  , addMoney
  ) where

newtype Currency (a :: Symbol) = Amount Number

rupiah :: Number -> Currency "rupiah"
rupiah = Amount

euro :: Number -> Currency "euro"
euro = Amount

addMoney :: ∀ a. Currency a -> Currency a -> Currency a
addMoney (Amount x) (Amount y) = Amount (x + y)

-- Contoh penggunaan
tenEuro = (euro 6.0) `addMoney` (euro 4.0)
tenRupiah = (rupiah 5.0) `addMoney` (rupiah 5.0)
```

Dengan pattern seperti ini, data invalid yang dihasilkan dari penjumlahan nominal dua buah mata uang yang berbeda pun dapat dicegah:

```hs
notValid = tenEuro `addMoney` tenRupiah
                              ^^^^^^^^^
-- |  Could not match type
-- |
-- |    "rupiah"
-- |
-- |  with type
-- |
-- |    "euro"
```

## Custom Kind
Pemodelan di atas masih belum bisa terlepas dari masalah string. Sekalipun ada di type level, pemodelan dengan string tetap rawan akan typo dan compiler tidak bisa menangkap kesalahan ini.

```hs
euroToRupiah :: Currency "euro" -> Currency "rupiahh_typo"
euroToRupiah (Amount eur) = Amount (eur * 15703.0)
```

Kita harus **mempersempit scope** karena lagi, string can contain anything: dengan cara membuat kind kita sendiri.

```hs
-- Buat custom kind
foreign import kind CurrencyK
foreign import data Rupiah :: CurrencyK
foreign import data Euro :: CurrencyK

-- Ubah dari `Symbol` ke `CurrencyK`
newtype Currency (a :: CurrencyK) = Amount Number

-- Bye-bye typo!
euroToRupiah :: Currency Euro -> Currency Rupiah
euroToRupiah (Amount e) = Amount (e * 15703.0)
```

Now the code looks safe already! 🎉🎉🎉

## Penutup
Ada beberapa hal penting yang bisa diambil di sini.

Yang pertama: having too much power is dangerous. Adanya type system yang berperan sebagai hansip selama ngoding dapat mencegah programmer dari kesalahan-kesalahan **kecil nan fatal**. Gak kebayang kalau masalah yang kelihatannya sepele ini bocor ke production dan merugikan banyak pihak.

Yang kedua, yang mungkin tidak terpikirkan sebelumnya: mengalihkan validation dari runtime ke compile time (dengan types) dapat menghemat banyak waktu debugging nantinya dan mengurangi penulisan test! Karena memang feedbacknya langsung terlihat: program compile atau tidak. Sedari awal kita **tidak pernah membuat runtime validation secuilpun** hanya untuk memastikan supaya Rupiah tidak tertukar dengan Euro. Namun code tetap safe dan hasil compile-nya pun gak kalah concise 😉

<div class="aligned-images">
  <img src="/uploads/purs-currency.png" alt="currency in purescript" />
  <img src="/uploads/js-currency.png" alt="generated currency code" />
</div>

Type system is there for a reason. Anggapan-anggapan bahwa type system membuat programmer tidak bebas memang ada benarnya, tapi perlu dibarengi dengan kesadaran bahwa menjadi _bebas_ tidak serta merta _terbebas_ dari apapun. Ada konsekuensinya. Ada harga yang harus dibayar. Dan semua keputusan dalam programming adalah trade-off.