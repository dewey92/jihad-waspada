---
title: "Contravariant Functor"
date: 2019-11-02T15:46:31+02:00
description: "Apa benar semua data dengan kind `Type → Type` adalah Functor? Bagaimana dengan type variable yang muncul di posisi negatif?"
images: ["/uploads/upside-down.jpg"]
image:
  src: "/uploads/upside-down.jpg"
  caption: Image by <a href="https://pixabay.com/users/Kranich17-11197573/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4017587">Kranich17</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4017587">Pixabay</a>
tags: ["purescript", "haskell", "functor", "functionalprogramming"]
categories: ["programming", "purescript"]
draft: false
---

Di artikel sebelumnya kita sudah membahas tentang [Functor]({{< ref "/post/code-reuse-berkaca-dari-functor.md" >}}) beserta motivasinya dari sisi code reusability. Artikel kali ini masih berbicara seputar Functor, namun dengan pembahasan lebih lanjut tentang ekplorasi idea **apakah setiap struktur data dengan [kind]({{< ref "/post/term-type-dan-kind-di-purescript.md" >}}) `Type -> Type` dapat otomatis dijadikan Functor**.

Dengan menganalisanya, kita akan melihat bagaimana Contravariant Functor lahir dari kombinasi anatara Functor dan function.

## Functor

Recall pemahaman kita soal definisi (Covariant) Functor.

```purs
class Functor f where
  map :: ∀ a b. (a -> b) -> f a -> f b
```

(Covariant) Functor adalah sebuah [type class]({{< ref "/post/kenalan-dulu-sama-type-class.md" >}}) dengan method tunggal `map`, dimana struktur data `f` harus memiliki kind `Type -> Type`. Dari type signature `map` bisa terlihat bahwa Functor tidak lain hanyalah sebuah wrapper dari type `a` yang &mdash; jika di-apply oleh function `a -> b` &mdash; akan mengubahnya menjadi wrapper dari type `b`.

Alternatively, Functor juga dapat dilihat sebagai "lifter" sebuah function: jika ada sebuah function `a -> b`, maka `map` akan mengubahnya menjadi `f a -> f b`.

```purs
map :: ∀ a b. (a -> b) -> (f a -> f b)
```

Di bawah ini akan kita eksplor bersama macam-macam struktur data dengan kind `Type -> Type` yang berpotensi dijadikan Functor, beserta implementasi `map`-nya jika memang memungkinkan.

### 1. Mention of Index

Ini adalah salah satu struktur data yang paling simple untuk dijadikan Functor, yaitu ia yang menggunakan type variable di salah satu inhabitant-nya, seperti `Identity`. Penggunaan type variable inilah yang dimaksud "mention of index".

```purs
data Identity x = MkIdentity x
-- `x` digunakan di sebelah kanan persamaan
```

Membuat instance Functor dari struktur data semacam ini sangatlah mudah. Mudah karena yang harus dilakukan hanyalah meng-apply `fn` terhadap data yang dibungkusnya.

```purs
instance functorId :: Functor Identity where
  map fn (MkIdentity x) = MkIdentity (fn x)
```

Pattern ini juga berlaku jika type variable muncul lebih dari satu kali. Atau untuk data constructor yang memiliki multiple arguments. Saya rasa _code speaks better than words_ 🙂

```purs
data Double x = MkDouble x x -- `x` muncul lebih dari sekali

instance functorDbl :: Functor Double where
  map fn (MkDouble a b) = MkDouble (fn a) (fn b)
--

data MultiArgs x = MkMultiArgs Int x String -- ada `x`, `String`, dan `Int`

instance functorMulti :: Functor MultiArgs where
  map fn (MkMultiArgs int x str) = MkMultiArgs int (fn a) str
```

### 2. Phantom Type

Phantom type adalah sebuah type constructor yang type variable-nya tidak muncul di sebelah kanan persamaan (no "mention of index").

```purs
data Phantom p = MkPhantom
```

Bisa dilihat type variable `p` tidak digunakan sama sekali oleh inhabitant-nya. Walaupun demikian, `Phantom` tetap memiliki kind `Type -> Type` yang cocok dengan Functor. Ada sesuatu yang unik di sini: `MkPhantom` tidak membungkus value apapun sehingga struktur data ini tidak akan pernah bisa ditransformasi, a.k.a static.

```purs
instance functorPhantom :: Functor Phantom where
  map _ MkPhantom = MkPhantom
```

Lain cerita jika ia memiliki index tambahan yang digunakan oleh inhabitant. Implementasi `map`-nya akan sama dengan "mention of index".

```purs
-- `p` is not used, but `x` is
data Phantom2 p x = MkPhantom2 x

instance functorPhantom2 :: Functor (Phantom2 p) where
  map fn (MkPhantom2 x) = MkPhantom (fn x)
```

Oh ya, data dengan kind lebih dari `Type -> Type` seperti Either dan Tuple kurang lebih juga menggunakan implementasi seperti ini, dimana partial apply harus dilakukan. Bisa dilihat di artikel saya sebelumnya [pada bagian Kind]({{< ref "/post/code-reuse-berkaca-dari-functor.md#kind" >}}).

### 3. Sum Types

Contoh Functor untuk sum type yang saya anggap cukup familiar adalah Maybe.

```purs
data Maybe a = Nothing | Just a
```

Untuk `Nothing`, strategi implementasi `map` bisa mengikuti caranya phantom type. Sementara `Just`, bisa mengikuti caranya "mention of index" 😉

```purs
instance functorMaybe :: Functor Maybe where
  map _ Nothing   = Nothing
  map fn (Just a) = Just (fn a)
```

### 4. Records

Implementasi functor untuk record hampir mirip dengan "mention of index" karena sejatinya record adalah product type dengan label.

```purs
data Rec x = MkRec { a :: x, b :: x, c :: String }

instance functorRec :: Functor Rec where
  map fn (MkRec { a, b, c }) = MkRec { a: fn a, b: fn b, c }
```

### 5. Recursive Type

List merupakan contoh struktur data rekursif paling umum yang bisa dijadikan Functor karena memiliki kind `Type -> Type`. Strategi implementasinya pun tidak jauh-jauh dari strategi "mention of index". Tinggal tambahkan recursive call saja 🙂

```purs
data List x = Nil | Cons x (List x)

instance functorList :: Functor List where
  map _ Nil = Nil
  map fn (Cons x rest) = Cons (fn x) (map fn rest)
--                                   ^^^^^^^^^^^^^
--                                   recursive call
```

Dengan demikian, struktur data rekursif lainnya seperti `Tree` juga dapat dengan mudah kita buatkan instance Functor-nya.

```purs
data Tree a = Leaf a | Branch a (Tree a) (Tree a)

instance functorTree :: Functor Tree where
  map fn (Leaf a)              = Leaf (fn a)
  map fn (Branch a left right) = Branch (fn a) (map fn left) (map fn right)
```

### 6. Function Return

Instance Functor juga dapat dibuat untuk sebuah data yang menjadi return value sebuah function. Let's say:

```purs
data Returning a = MkReturning (String -> a)
```

Secara intuitif, kita harus menjalankan fungsi `String -> a` terlebih dahulu (dengan menyuplai sebuah string) untuk mendapatkan `a`, yang nantinya akan di-apply dengan fungsi `fn :: a -> b` agar `a` menjadi `b`. Setelah menjadi `b`, kita bungkus lagi deh dengan function `String -> b`. Neat 😉

```purs
instance functorReturning :: Functor Returning where
  map fn (MkReturning strToA) = MkReturning (\str -> fn (strToA str))
  -- `strToA` :: String -> a
  -- `strToA str` :: a
  -- `fn` :: a -> b
  -- `fn (strToA str)` :: b
  -- `(\str -> fn (strToA str))` :: String -> b


-- Yang bisa disederhanakan menjadi function composition biasa
instance functorReturning :: Functor Returning where
  map fn (MkReturning strToA) = MkReturning (fn <<< strToA)
```

---
<span></span>
<section class="stripes">

### ℹ️ FYI ℹ️

Dengan asumsi bahwa `F` memiliki kind `Type -> Type`, compiler Purescript sebenarnya dapat membuat instance Functor secara otomatis, tanpa mengharuskan programmer untuk menuliskan implementasinya.

```purs
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

Dengan kata lain, instance Functor dari keenam contoh di atas dapat dihapus dan cukup menuliskannya seperti ini:

```purs
derive instance functorPhantom   :: Functor Phantom
derive instance functorId        :: Functor Identity
derive instance functorMulti     :: Functor MultiArgs
derive instance functorMaybe     :: Functor Maybe
derive instance functorRec       :: Functor Rec
derive instance functorList      :: Functor List
derive instance functorTree      :: Functor Tree
derive instance functorReturning :: Functor Returning
```

Tidak ada behaviour yang berubah, semua tetap sama 😃🎉

</section>

---

## Contravariant Functor

Sebelum melanjutkan ke pembahasan Contravariant Functor, saya punya sebuah quiz kecil yang masih berhubungan dengan contoh nomor 6 di atas: Bagaimana menerapkan instance functor dari `Consuming` dimana **posisi index ada di function argument**?

```purs
data Consuming a = MkConsuming (a -> String)
```

### Intuisi

Mari kita telaah step by step. Argument pertama pada `map` memiliki type signature `a -> b`. Sekilas tipe Consuming ini memberikan kesan bahwa ia pasti memiliki type `(a -> String) -> (b -> String)` ketika di-map. Betul apa betul?

```purs
map :: (a -> b) -> (f a -> f b)
-- `fn` :: a -> b
-- `f a` ~ MkConsuming (a -> String)
-- `f b` ~ MkConsuming (b -> String)
```

Namun setelah dipikir-pikir, **fungsi `fn :: a -> b` mustahil di-compose dengan `(a -> String)`**! Mau jalanin `fn` dulu baru `f a`? Gak bisa. Atau jalanin `f a` dulu baru `fn`? Gak bisa juga!

```purs
-- 1. `fn` then `f a`?
(a -> b) -> (a -> String) -- `b` gak sama dengan `a` ❌
      ⎣______⎦


-- 2. `f a` then `fn`?
(a -> String) -> (a -> b) -- `String` gak sama dengan `a` ❌
         ⎣________⎦
```

Nah karena kita ingin memanipulasi tipe `a` dengan `fn`, opsi nomor 2 ini sudah pasti gak valid (malah String yang dimanipulasi 😕). Dan satu-satunya jalan untuk memanipulasi tipe `a` adalah dengan menyuplai sebuah fungsi `b -> a`.

```purs
(b -> a) -> (a -> String) -- `a` sekarang sama dengan `a`! ✅
      ⎣______⎦

-- Dengan funtion composition biasa, notasi ini bisa direduksi menjadi
(b -> String)
```

Dan akhirnya, dengan fungsi `b -> a` kita bisa mengubah `(a -> String)` menjadi `(b -> String)`! Ketika fungsi kebalik ini dijadikan type class, maka lahirlah saudara baru Functor, yang bernama Contravariant Functor.

```purs
class Contravariant f where
  cmap :: ∀ a b. (b -> a) -> f a -> f b

-- Perbandingannya dengan fungsi `map`.
map  :: (a -> b) -> f a -> f b
cmap :: (b -> a) -> f a -> f b
```

Balik ke soal quiz tadi. Apakah tipe data `Consuming` bisa menjadi Functor? Sayangnya kita harus menjawab tidak. Jawaban ini sekaligus membantah idea di awal artikel yang menyebutkan bahwa setiap data yang memiliki kind `Type -> Type` adalah sebuah Functor.

Walaupun `Consuming` tidak bisa menjadi Functor, ia masih bisa menjelma jadi Contravariant Functor 🙃

```purs {hl_lines=[1,2]}
instance contraConsuming :: Contravariant Consuming where
  cmap fn (MkConsuming aToStr) = MkConsuming (fn >>> aToSsr)

-- `Returning` for reference
instance functorReturning :: Functor Returning where
  map fn (MkReturning strToA) = MkReturning (fn <<< strToA)
```

Perbedaan `Consuming` dengan `Returning` sangat simple: pada `Consuming` fungsi transformasi `fn` dijalankan terlebih dahulu baru kemudian value di dalamnya. Sedangkan `Returning` sebaliknya, jalankan dulu value di dalamnya lalu transformasikan hasilnya dengan `fn`.

Ilustrasi di bawah ini semoga membantu.

```purs
   contra f
 ⎡‾‾‾‾‾‾‾‾‾‾‾⎤
fn >>> valueAsFunction >>> fn
               ⎣____________⎦
                  functor
```

### Penerapan

Penerapan Contravariant Functor memang tidak sebanyak saudara tirinya, Functor. Namun mereka masih berbagi visi yang sama yaitu untuk menghasilkan code yang reusable. Contohnya pada fungsionalitas sorting.

Anggap ada object Person dan program dapat melakukan sorting berdasarkan `name` dan `age`. Kita mulai dengan solusi tanpa Contravariant Functor.

```purs {hl_lines=["7-8","10-11"]}
type Person = { name :: String, age :: Int }

-- Factory function untuk sorting
mkCompare :: forall a b. Ord b => (a -> b) -> a -> a -> Ordering
mkCompare fn a b = compare (fn a) (fn b)

cmpByName :: Person -> Person -> Ordering
cmpByName = mkCompare (_.name)

cmpByAge :: Person -> Person -> Ordering
cmpByAge = mkCompare (_.age)

persons :: Array Person
persons = [
  { name: "A", age: 22 },
  { name: "B", age: 21 },
  { name: "C", age: 23 },
]
--

λ> sortBy cmpByAge persons
[
  { name: "B", age: 21 },
  { name: "A", age: 22 },
  { name: "C", age: 23 }
]
```

Namun ternyata fungsionalitas sorting Person ini gak hanya digunakan di satu tempat saja. Let's say ia juga digunakan ketika handling HTTP Request yang mengandung object Person.

```purs
type PersonRequestBody = {
  person :: Person,
  ...
}

cmpByPersonName :: PersonRequestBody -> PersonRequestBody -> Ordering
cmpByPersonName = mkCompare (_.person.name)

cmpByPersonAge :: PersonRequestBody -> PersonRequestBody -> Ordering
cmpByPersonAge = mkCompare (_.person.age)
```

Cukup simple dan straightforward. Namun ada beberapa bagian code yang terlihat redundan seperti record accessor dan type signature. Contravariant Functor dapat membantu pengabstraksian konsep sorting ini dan membuatnya menjadi lebih composable: satu fungsi dibangun di atas fungsi lain.

Langkah pertamanya, dengan meng-capture idea sorting ini ke sebuah struktur data sendiri (kita namakan `Comparison`) lalu membuat instance Contravariant-nya.

```purs
newtype Comparison a = Comparison (a -> a -> Ordering)

instance contraComp :: Contravariant Comparison where
  cmap fn (Comparison c) = Comparison (\x y -> c (fn x) (fn y))

-- utility functions

getComparison :: ∀ a. Comparison a -> a -> a -> Ordering
getComparison (Comparison fn) = fn

defaultCompare :: ∀ a. Ord a => Comparison a
defaultCompare = Comparison compare
```

Bagian setelah ini cukup menarik karena fungsi-fungsi komparasi untuk `Person` dan `PersonRequestBody` dibuat dengan composition biasa, memungkinkan kita untuk me-reuse fungsi-fungsi yang sudah ada.

```purs
byName :: Comparison Person
byName = cmap (_.name) defaultCompare

byAge :: Comparison Person
byAge = cmap (_.age) defaultCompare

byPersonName :: Comparison PersonRequestBody
byPersonName = cmap (_.person) byName

byPersonAge :: Comparison PersonRequestBody
byPersonAge = cmap (_.person) byAge
--

λ> sortBy (getComparison byAge) persons
[
  { name: "B", age: 21 },
  { name: "A", age: 22 },
  { name: "C", age: 23 }
]
```

Abstraksi Comparison ini bisa ditemukan di package [purescript-contravariant](https://github.com/purescript/purescript-contravariant/blob/cb69db0253c2e2ed3fef784dad58f3418a8ee834/src/Data/Comparison.purs#L14-L15).

Saya rekomendasikan bagi temen-temen yang ingin tahu lebih lanjut tentang Contravariant untuk menonton video dari Mas George Wilson.

{{< youtube JZPXzJ5tp9w >}}

---

## What's Next

Contravariant Functor bisa jadi tidak setenar Functor karena penggunaannya yang memang sedikit. Ia baru akan terlihat ketika **dikombinasikan** dengan (Covariant) Functor dan Bifunctor. Bifunctor yang contravariant di argument pertamanya dan covariant di argumen keduanya akan membentuk jenis Functor baru bernama _Profunctor_.

Bifunctor insyallah akan kita bahas pada artikel selanjutnya. Stay tuned 😉
