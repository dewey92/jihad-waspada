---
title: "Term, Type, dan Kind di Purescript"
date: 2019-08-28T01:00:48+02:00
description: "Masih ada dunia lain di atas types: dunia kind"
images: ["/uploads/aurora.jpg"]
image:
  src: "/uploads/aurora.jpg"
  caption: Image by <a href="https://pixabay.com/users/Photo-View-7444623/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3847784">Tommy Andreassen</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3847784">Pixabay</a>
tags: ["purescript", "types"]
categories: ["programming", "purescript", "type system"]
draft: false
---

Ini artikel bakal receh banget, tapi mungkin bakal sering saya jadikan acuan atau rujukan di artikel-artikel selanjutnya. Jadi yaudah, ditulis aja dulu 😄

## Term
Mulai dari term. Term adalah semua value yang biasa kita lihat sehari-hari. Ada `true`, `false`, `"jihad"`, `5`, `CustomData`. Apapun itu.

Tiap-tiap term pasti punya type. `true` bertipe `Boolean`, `"jihad"` bertipe `String`, `5` bertipe `Int`, dan lain sebagainya. Gak ada term yang gak punya type.

## Type dan Kind
Jika term kita ibaratkan sebagai anak dan type sebagai orangtua-nya, kind bisa diibaratkan sebagai kakek-neneknya. Jadi type sendiri sebenarnya punya type.

Di Purescript, type-nya type ini disebut _kind_ dan biasa ditulis dengan notasi `Type`. Contoh-contoh berikut saya harap bisa menjawab mengapa `Type` digunakan untuk mengekspresikan kind.

### Boolean
Type `Boolean` terdiri dari dua buah term: true dan false.

```no-code
Kind |           Type
                   ↑
Type |          Boolean
              ↗         ↖
Term |      true      false
```

### String
Term yang bertipe string dapat berupa apa saja!

```no-code
Kind |              Type
                      ↑
Type |              String
              ↗       ↑       ↖ ️
Term |     "jihad" "dzikri" "waspada"
```

Kind dari `Boolean` dan `String` sama-sama `Type`. Kenapa? Untuk menjawab pertanyaan ini, saya mau nanya balik dulu: "Apa sih Boolean _itu_?". Pasti jawabannya, "Boolean adalah sebuah type". Sama kayak string: "Apa sih String _itu_?". Jawabannya, "String adalah sebuah type".

Gak heran disebutnya `Type` 😛

### Type dan Data Constructor
Misal ada sebuah data coordinate:

```purs
data Coord = MkCoord Int Int

> :t MkCoord
> Int -> Int -> Coord

> :t MkCoord 2
> Int -> Coord

> :t MkCoord 2 3
> Coord
```

`MkCoord` adalah sebuah data constructor yang menerima type `Int` dan `Int`, yang menghasilkan sebuah value bertipe `Coord`. Dengan kata lain, `MkCoord` memiliki type `Int -> Int -> Coord`.

```no-code
Kind |                            Type
                        ↗          ↑             ↖
Type |  Int -> Int -> Coord   Int -> Coord      Coord
              ↑
Term |     MkCoord              MkCoord 2     MkCoord 2 3
```

Purescript secara default sudah support currying. Dan gak cuman buat data constructor, untuk type constructor pun juga bisa currying.

```purs
data Maybe a = Just a | Nothing

> :k Maybe
> Type -> Type

> :k Maybe Int
> Type
```

Notasi `Type -> Type` berbunyi: "kalau kamu kasih aku sebuah type, maka aku akan mengembalikan sebuah type."

Oh ya perlu saya sebutkan bahwa sebuah type parameter (seperti `a` dalam `Maybe a`) memiliki kind `Type` by default kecuali secara eksplisit dinyatakan sebagai kind lainnya (seperti Symbol).

```no-code
Kind |  Type -> Type             Type
             ↑               ↗ ️         ↖
Type |     Maybe       Maybe a          Maybe Int
                          ↑            ↗         ↖
Term |       -         Nothing     Just 5  (Nothing :: Maybe Int)
```

### Symbol
Symbol adalah kind dari type level string. Jadi string di Purescript itu bisa hidup di dua alam: term level dan type level.

```purs
type Jihad = "jihad"
type Waspada = "waspada"

> :k Jihad
> Symbol

> :k Waspada
> Symbol
```

```no-code
Kind |           Type                 Symbol
                  ↑                 ↗        ↖
Type |          String          "jihad"   "waspada"
              ↗        ↖
Term |    "jihad"   "waspada"      -          -
```

Nah ini adalah salah satu contoh kasus dimana type tidak melulu harus "bertipe" `Type`. Bahkan kita bisa membuat kind kita sendiri di Purescript!

### Custom Kind
Untuk membuat custom kind di Purescript, kita musti menggunakan keyword `foreign import` walaupun sebenarnya tidak ada data yang di-import.

```purs
foreign import kind Boolean
foreign import data True :: Boolean
foreign import data False :: Boolean

> :k True
> Boolean

> :k False
> Boolean
```

```no-code
Kind |          Boolean
              ↗         ↖
Type |      True       False

Term |       -           -
```

Bisa dilihat bahwa pada contoh Symbol dan Custom Kind ini, **tidak ada term yang bisa merepresentasikan type-type tersebut**. Artinya penggunaannya memang hanya diperuntukkan untuk type-level programming saja. Tak kasih contoh kecil:

```purs
import Prelude
import Data.Symbol (class IsSymbol, SProxy(..), reflectSymbol)

foreign import kind IyaNggak
foreign import data Iya :: IyaNggak
foreign import data Nggak :: IyaNggak

class IsGanteng (orang :: Symbol) (bool :: IyaNggak)

instance jihadGanteng     :: IsGanteng "jihad" Iya
instance squidwardGanteng :: IsGanteng "squidward" Iya
instance kamuGanteng      :: IsGanteng "kamu" Nggak

enterRoom :: forall a.
  IsSymbol a =>
  IsGanteng a Iya =>
  SProxy a -> String
enterRoom sym = "Selamat datang " <> (reflectSymbol sym)
```

Perhatikan type signature fungsi `enterRoom`. Fungsi ini memiliki constraint `IsGanteng a Iya` yang artinya hanya orang-orang ganteng aja yang bisa mengakses fungsi `enterRoom`.

```purs
bolehMasuk  = enterRoom (SProxy :: _ "jihad")
bolehMasuk2 = enterRoom (SProxy :: _ "squidward")
```

Kalo ada yang coba-coba masuk selain `"jihad"` dan `"squidward"`, compiler bakal kasih error dan menolak untuk compile.

```purs
takBoleh = enterRoom (SProxy :: _ "kamu")
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
-- |  No type class instance was found for
-- |
-- |    IsGanteng "kamu" True

```

{{< figure src="https://media.giphy.com/media/7YeguV6Ia9lfO/giphy.gif" alt="yeah right" caption="squidward ganteng" class="fig-center img-60" >}}

## Penutup
Sekarang sudah tau kan bedanya term, type, dan kind. Tapi pertanyaan besar muncul: terus kenapa gitu lho? Kenapa juga harus ada kind?

Di artikel saya yang lalu tentang [HKT di Typescript]({{< ref "/post/generic-di-atas-generic-higher-kinded-type.md" >}}), kita bisa menarik kesimpulan bahwa di bahasa-bahasa yang tidak memiliki klasifikasi type (kind) seperti Typescript, semua type variable diasumsikan memiliki kind `Type`. Walhasil konsep abstraksi seperti Functor dan Monad akan sulit diekspresikan oleh type system.

Kind juga membuka kesempatan bagi programmer untuk melakukan type-level programming (seperti yang saya demokan tadi) yang bisa digunakan sebagai **constraint** agar invalid data/state saat runtime dapat terhindarkan.
