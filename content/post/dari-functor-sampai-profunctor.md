---
title: "Dari Functor Sampai Profunctor"
date: 2019-10-23T21:12:31+02:00
description: ""
images: [""]
image:
  src: ""
  caption: ""
tags: [""]
categories: [""]
draft: true
---

Suatu saat pernah terlintas di timeline twitter saya sebuah tweet menarik tentang ide bahwa untuk setiap struktur data yang memiliki [kind]({{< ref "/post/term-type-dan-kind-di-purescript.md" >}}) `Type -> Type` dapat juga memiliki instance Functor. Tweet ini sedikit banyak mendorong saya untuk googling kecil-kecilan apa benar demikian. Dan untuk lebih mengokohkan pemahaman saya soal Functor dan sepupu-sepupunya, saya putuskan untuk menyusun artikel receh ini dengan harapan temen-temen juga dapat memahami konsep Functor secara matang.

## Functor

Recall pemahaman kita soal definisi Functor.

```hs
class Functor f where
  map :: ∀ a b. (a -> b) -> f a -> f b
```

Functor adalah sebuah [type class]({{< ref "/post/kenalan-dulu-sama-type-class.md" >}}) dengan method tunggal `map`, dimana `f` (struktur data yang boleh memiliki instance Functor) harus memiliki kind `Type -> Type`.

Dari type signature, kita bisa memahami bahwa Functor tidak lain hanyalah sebuah wrapper dari type `a` yang &mdash; jika di-apply oleh function `a -> b` &mdash; akan menjadi wrapper dari type `b`.

Dilihat dari sudut pandang lain, Functor juga dapat dipahami sebagai "lifter" sebuah function: jika ada sebuah function `a -> b`, maka `map` akan mengubahnya menjadi `f a -> f b`.

```hs
map :: ∀ a b. (a -> b) -> (f a -> f b)
```

Contoh implementasi Functor sudah baaaanyak sekali, ada `Array`, `Maybe`, `Either`, `Tuple`, dan lain sebagainya. Di bawah ini saya sertakan beberapa contoh dasar bagaimana kita dapat membuat instance Functor dari struktur data dengan shape yang berbeda-beda.

### 1. Phantom Type

Phantom type adalah sebuah type constructor yang type variable-nya tidak muncul di sebelah kanan persamaan.

```hs
data Phantom p = MkPhantom
```

Bisa dilihat type variable `p` tidak digunakan sama sekali oleh inhabitant-nya. Walaupun demikian, `Phantom` tetap memiliki kind `Type -> Type` yang berpotensi untuk dijadikan Functor.

```hs
instance functorPhantom :: Functor Phantom where
  map fn MkPhantom = MkPhantom
```

`MkPhantom` tidak membungkus data apapun, sehingga tidak ada sesuatu yang bisa ditransformasi. Fungsi `fn` pun menjadi tidak berguna. Oleh karenanya `MkPhantom` langsung dikembalikan. This is a valid functor eventhough nothing is changed whatsoever.

### 2. Mention of Index

Struktur data ini adalah struktur data yang menurut saya pribadi paling simple dan mudah dimengerti untuk memahami Functor, yaitu ia yang menggunakan type variable di salah satu inhabitant-nya, seperti `Identity`.

```hs
data Identity x = MkIdentity x

instance functorId :: Functor Identity where
  map fn (MkIdentity x) = MkIdentity (fn x)
```

Membuat instance Functor dari struktur data semacam ini sangat sangatlah mudah. Mudah karena yang harus dilakukan hanyalah meng-apply `fn` terhadap data yang dibungkusnya.

Pattern ini juga berlaku jika type variable muncul lebih dari satu kali. Atau untuk data constructor yang memiliki multiple arguments. Saya rasa _code speaks better than words_ 🙂

```hs
data Double x = MkDouble x x -- `x` muncul lebih dari sekali

instance functorDbl :: Functor Double where
  map fn (MkDouble a b) = MkDouble (fn a) (fn b)
--

data MultiArgs x = MkMultiArgs Int x String -- ada `x`, `String`, dan `Int`

instance functorMulti :: Functor MultiArgs where
  map fn (MkMultiArgs int x str) = MkMultiArgs int (fn a) str
```

### 3. Sum Types

Contoh Functor untuk sum type yang saya anggap cukup familiar adalah Maybe.

```hs
data Maybe a = Nothing | Just a
```

Untuk `Nothing`, strategi implementasi functor bisa mengikuti caranya phantom type. Sementara `Just`, bisa mengikuti caranya "mention of index" barusan 😉

```hs
instance functorMaybe :: Functor Maybe where
  map _ Nothing   = Nothing
  map fn (Just a) = Just (fn a)
```

### 4. Records

Implementasi functor untuk record hampir mirip dengan "mention of index" karena sejatinya record adalah product type dengan label.

```hs
data Rec x = MkRec { a :: x, b :: x, c :: String }

instance functorRec :: Functor Rec where
  map fn (MkRec { a, b, c }) = MkRec { a: fn a, b: fn b, c }
```

### 5. Recursive Type

List merupakan contoh struktur data rekursif paling umum yang bisa dijadikan Functor karena memiliki kind `Type -> Type`. Strategi implementasinya pun tidak jauh-jauh dari strategi "mention of index". Tinggal tambahkan recursive call saja 🙂

```hs
data List x = Nil | Cons x (List x)

instance functorList :: Functor List where
  map _ Nil = Nil
  map fn (Cons x rest) = Cons (fn x) (map fn rest)
--                                   ^^^^^^^^^^^^^
--                                   recursive call
```

Dengan demikian, struktur data rekursif lainnya seperti `Tree` juga dapat dengan mudah kita buatkan instance Functor-nya.

```hs
data Tree a = Node a | Branch (Tree a) (Tree a)

instance functorTree :: Functor Tree where
  map fn (Node a)            = Node (fn a)
  map fn (Branch left right) = Branch (map fn left) (map fn right)
```

### 6. Function Return

Instance Functor juga dapat dibuat untuk sebuah data yang menjadi return value sebuah function. Lets's say:

```hs
data Returning a = MkReturning (String -> a)
```

Secara intuitif, kita harus menjalankan fungsi `String -> a` terlebih dahulu (dengan menyuplai sebuah string) untuk mendapatkan `a`, yang nantinya akan di-apply dengan fungsi `fn :: a -> b` agar `a` menjadi `b`. Setelah menjadi `b`, kita bungkus lagi deh dengan function `String -> b`. Neat 😉

```hs
instance functorReturning :: Functor Returning where
  map fn (MkReturning strToA) = MkReturning (\str -> fn (strToA str))
  -- `strToA` :: String -> a
  -- `strToA str` :: a
  -- `fn` :: a -> b
  -- `fn (strToA str)` :: b
  -- `(\str -> fn (strToA str))` :: String -> b
```

Posisi `a` yang **muncul di return type sebuah function** biasa dinamakan posisi Covariant. Begitu juga dengan type variable pada contoh-contoh lain di atas, mereka semua berada di posisi Covariant.

```hs
data Identity x = MkIdentity x -- x is in covariant position
data Returning a = MkReturning (String -> a) -- a is in covariant position
```

---
<span></span>
<section style="background: repeating-linear-gradient(
  135deg,
  white,
  white 40px,
  #f4f4f4 40px,
  #f4f4f4 80px
)">

### ℹ️ FYI ℹ️

Dengan asumsi bahwa `F` memiliki kind `Type -> Type`, compiler Purescript sebenarnya dapat membuat instance Functor secara otomatis, tanpa mengharuskan programmer untuk menuliskan implementasinya.

```hs
-- | Alih-alih menulis instance Functor secara manual dengan
-- |
-- | ```
-- | instance functorF :: Functor F where
-- |   map f (F) = ...
-- | ```
-- |
-- | Programmer dapat memanfaatkan bantuan compiler
-- | dengan keyword `derive instance`

derive instance functorF :: Functor F
```

Lho gimana bisa? 🤔 Ternyata compiler Purescript memiliki algortima khusus (yang dikembangkan oleh [Pakde Liam Goodacre](https://liamgoodacre.github.io/purescript/functor/deriving/2017/01/23/deriving-functor.html) semenjak release versi 0.10.4) untuk menentukan tipe data apa saja yang bisa secara otomatis dijadikan Functor.

Dengan kata lain, instance Functor dari keenam contoh di atas dapat kita hapus lalu kita instruksikan compiler agar men-generate instance untuk kita secara cuma-cuma!

```hs
derive instance functorPhantom   :: Functor Phantom
derive instance functorId        :: Functor Identity
derive instance functorMulti     :: Functor MultiArgs
derive instance functorMaybe     :: Functor Maybe
derive instance functorRec       :: Functor Rec
derive instance functorList      :: Functor List
derive instance functorTree      :: Functor Tree
derive instance functorReturning :: Functor Returning
```

Tidak ada behaviour yang berubah, semua tetap sama 😃

<small>* Excuse me for this ugly stripes</small>

</section>

---

## Contravariant Functor

Sebelum melanjutkan ke pembahasan Contravariant Functor (yang selebihnya akan disingkat dengan Cofunctor), saya punya sebuah quiz kecil yang masih berhubungan dengan contoh nomor 6 di atas: Bagaimana menerapkan instance functor dari struktur data `Consuming` berikut?

```hs
data Consuming a = MkConsuming (a -> String)

-- Sebagai referensi dan biar gak capek-capek scroll,
-- tak tuliskan lagi tipe data `Returning`
data Returning a = MkReturning (String -> a)
```

### Intuisi

Mari kita telaah step by step. Argument pertama pada `map` memiliki type signature `a -> b`. Sekilas tipe Consuming ini memberikan kesan bahwa ia pasti memiliki type `(a -> String) -> (b -> String)` ketika di-map. Betul apa betul?

```hs
map :: (a -> b) -> (f a -> f b)
-- `fn` :: a -> b
-- `f a` ~ MkConsuming (a -> String)
-- `f b` ~ MkConsuming (b -> String)
```

Namun setelah dipikir-pikir, **fungsi `a -> b` mustahil bisa dipanggil dengan `(a -> String)`**! Mau jalanin `fn` dulu baru `f a`? Gak bisa. Atau jalanin `f a` dulu baru `fn`? Gak bisa juga!

```hs
-- 1. `fn` then `f a`?
(a -> b) -> (a -> String) -- `b` gak sama dengan `a` ❌
      ⎣⎽⎽⎽⎽⎽⎽⎦


-- 2. `f a` then `fn`?
(a -> String) -> (a -> b) -- `String` gak sama dengan `a` ❌
         ⎣⎽⎽⎽⎽⎽⎽⎽⎽⎦
```

Nah karena kita ingin memanipulasi tipe `a`, opsi nomor 2 ini sudah pasti gak valid. Dan satu-satunya jalan untuk memanipulasi tipe `a` adalah dengan menyuplai sebuah fungsi `b -> a`.

```hs
(b -> a) -> (a -> String) -- `a` sekarang sama dengan `a`! ✅
      ⎣⎽⎽⎽⎽⎽⎽⎦

-- Dengan funtion composition biasa, notasi ini bisa direduksi menjadi
(b -> String)
```

Dan akhirnya, dengan fungsi `b -> a` kita bisa mengubah `(a -> String)` menjadi `(b -> String)`! Ketika fungsi kebalik ini digunakan oleh sebuah functor, maka lahirlah saudara baru, yang bernama Contravariant Functor.

```hs
class Contravariant f where
  cmap :: ∀ a b. (b -> a) -> f a -> f b

-- Silakan bandingkan dengan fungsi `map` biasa.
map  :: (a -> b) -> f a -> f b
cmap :: (b -> a) -> f a -> f b
```

Balik ke soal quiz tadi. Apakah tipe data `Consuming` bisa menjadi Functor? Sayangnya kita harus menjawab tidak. Jawaban ini sekaligus membantah idea di awal artikel yang menyebutkan bahwa setiap data yang memiliki kind `Type -> Type` adalah sebuah Functor.

Nah walaupun `Consuming` tidak bisa menjadi Functor, ia masih bisa menjelma menjadi Cofunctor!

```hs
instance contraConsuming :: Contravariant Consuming where
  cmap fn (Consuming aToStr) = Consuming (fn >>> aToSsr)
```

### Kegunaan
TODO