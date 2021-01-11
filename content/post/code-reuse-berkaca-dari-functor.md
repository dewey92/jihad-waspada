---
title: "Code Reuse — Berkaca dari Functor"
date: 2019-10-29T21:37:42+01:00
description: "Pengenalan konsep Functor dari sisi code reusability dengan Purescript"
images: ["/uploads/functor-reuse.jpg"]
image:
  src: "/uploads/functor-reuse.jpg"
  caption: Image by <a href="https://pixabay.com/users/annca-1564471/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3289812">annca</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3289812">Pixabay</a>
tags: ["purescript", "haskell", "functor", "functionalprogramming"]
categories: ["programming", "purescript", "functional programming"]
draft: false
---

Ada banyak cara untuk memahami Functor. Yang paling umum dan banyak dijumpai ialah dengan penjelasan wrap-unwrap data, seperti di artikel [Bungkus Kacang](https://medium.com/beingprofessional/understanding-functor-and-monad-with-a-bag-of-peanuts-8fa702b3f69e) atau [ilustrasinya mas Adit](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html). Di artikel ini saya mencoba menjelaskannya dari sudut pandang lain: code reuse.

---

## Motivasi

Mari kita mulai dengan sesuatu yang simple: sebuah angka. Lalu kita lakukan transformasi pada angka tersebut dengan fungsi `incr` dan `decr`.

```hs
incr :: Int -> Int
incr n = n + 1

decr :: Int -> Int
decr n = n - 1

--

λ> incr 5
6
λ> decr 10
9
```

Easy. Namun apa jadinya jika angka tersebut dimasukkan atau dibungkus **ke dalam suatu data structure** seperti Array? Apa ia masih bisa di-increment atau decrement?

```hs
incrArray :: Array Int -> Array Int
decrArray :: Array Int -> Array Int
```

Ekspektasi kita adalah ketika `incrArray` dijalankan, angka yang ada di dalam Array harus bertambah satu. `incrArray [5]` harus menghasilkan `[6]`. Demikian pula dengan `decrArray` ketika dijalankan, angka yang ada di dalam Array harus berkurang satu. `decrArray [10]` harus mengembalikan `[9]`.

Kalo maunya begitu, apa fungsi `incr` dan `decr` masih bisa digunakan?

Banyak pertanyaan untuk dijawab 😄

## Reusability

Sebagai seorang programmer kita terbiasa melihat pola dalam code dan membuat abstraksinya agar code tidak redundan. Tentunya kita tidak ingin membuat fungsionalitas increment-decrement dari awal lagi hanya untuk membuatnya bekerja dengan Array. Kita ingin `incr` dan `decr` **reusable**.

Salah satu upaya agar code menjadi lebih reusable adalah dengan memanfaatkan Higher-Order Function.

```hs
-- Implementasinya kita kosongkan dulu.
-- Kita asumsikan `arr` di-loop dan apply `editFn` di setiap elementnya
transformArray :: (Int -> Int) -> Array Int -> Array Int
transformArray editFn arr = ...
```

Fokus kita ada di type signature: `transformArray` menerima fungsi `(Int -> Int)` di argument pertamanya, cocok dengan `incr` dan `decr` yang juga bertipe `Int -> Int`! Bisa langsung disubstitusi deh.

```hs
incrArray :: Array Int -> Array Int
incrArray = transformArray incr

decrArray :: Array Int -> Array Int
decrArray = transformArray decr
```

Walaupun `incr` dan `decr` berhasil di-reuse, `transformArray` ini belum cukup reusable: **ia hanya menerima fungsi bertipe `Int -> Int`**. Fungsi dengan type signature lain seperti `isEven :: Int -> Boolean` atau `showInt :: Int -> String` tidak bisa digunakan.

Lebih parahnya lagi, fungsi `transformArray` hanya bisa memproses `Array Int`! Sedangkan tau sendiri, Array itu termasuk struktur data yang generic, bisa terisi oleh berbagai macam tipe data: `Array String`, `Array Boolean`, `Array User` dan lain sebagainya. Kita ingin fungsi ini lebih general dan reusable.

Di sinilah generic mulai berperan. Kita harus mengubah type signature yang terlalu spesifik terhadap Int dengan generics.

```hs
-- Before
transformArray :: (Int -> Int) -> Array Int -> Array Int

-- After
transformArray :: ∀ a b. (a -> b) -> Array a -> Array b
```

Perubahan dari concrete type ke generics memperluas lingkup function, menjadikannya jauh lebih reusable. Lihat saja contoh-contoh berikut, kita tak lagi terikat dengan `Int -> Int` dan bisa bekerja dengan berbagai macam Array 🎉🎉🎉

```hs
length :: String -> Int
asciiCode :: Char -> Int
isEven :: Int -> Boolean

--

λ> transformArray length ["jihad", "waspada"]
[5, 7]
λ> transformArray asciiCode ['A', 'B', 'a', 'b']
[65, 66, 97, 98]
λ> transformArray isEven [1, 2, 3, 4]
[false, true, false, true]
```

{{< figure src="https://media.giphy.com/media/kBZBlLVlfECvOQAVno/200w_d.gif" alt="proud of myself" caption="tepuk tangan dulu" class="fig-center img-60" >}}

---

## Bentuk Lain

Kita sudah berhasil "menakhlukkan" Array. Saatnya membahas struktur data generic lain yang agak menantang. Let's talk about Tree.

```hs
data Tree a = Leaf a | Branch a (Tree a) (Tree a)

treeExample :: Tree Int
treeExample = Branch 1
  (Leaf 2)
  (Branch 3 (Leaf 4) (Leaf 5))

   <1>
  /   \
<2>   <3>
     /   \
   <4>   <5>
```

Tree sama seperti Array, ia bisa menampung berbagai macam jenis data: `Tree Int`, `Tree String`, `Tree Boolean`, dll. Bisa ditransformasi juga? Harusnya sih bisa. Karena kita sudah belajar bagaimana membuat fungsi yang reusable pada Array, kita pun juga ingin agar fungsi transformasi pada struktur data Tree ini reusable.

```hs
transformTree :: ∀ a b. (a -> b) -> Tree a -> Tree b
transformTree editFn tree = ...

--

λ> transformTree incr treeExample
Branch 2
  (Leaf 3)
  (Branch 4 (Leaf 5) (Leaf 6))
```

Fokus pada type signature, kita melihat pola yang **sangat identik** pada fungsi `transformTree` dan `transformArray`. Mereka identik dalam 2 hal:

1. Behaviour (sama-sama sebagai fungsi transformasi)
2. Type signature

Mungkin ketika suatu saat nanti ada struktur data lain (misal Queue), kita juga akan membuat fungsi `transformQueue` dengan type signature yang sama.

```hs
transformArray :: (a -> b) -> Array a -> Array b
transformTree  :: (a -> b) -> Tree a  -> Tree b
transformQueue :: (a -> b) -> Queue a -> Queue b
...
...
transformBlabla :: (a -> b) -> Blabla a -> Blabla b
```

`Array`, `Tree`, `Queue`, `Blabla` semua ini bisa direpresentasikan dengan variable `t`.

```hs
transform :: (a -> b) -> t a -> t b
```

Sehingga jadilah sebuah fungsi generic baru untuk transformasi value di dalam suatu struktur data 🙂. Mari buatkan juga [Type Class]({{< ref "kenalan-dulu-sama-type-class.md" >}})-nya agar `t` dapat disubstitusi.

```hs {hl_lines=[1,2]}
class Transformable t
  transform :: ∀ a b. (a -> b) -> t a -> t b

instance transArray :: Transformable Array where
  transform = transformArray

instance transTree :: Transformable Tree where
  transform = transformTree

instance transQueue :: Transformable Queue where
  transform = transformQueue

...
...

instance transBlabla :: Transformable Blabla where
  transform = transformBlabla
```

Class Transformable sudah dibuat. Fungsi `transform` akhirnya bisa dipanggil oleh Array, Tree, dan Queue.

```hs
λ> transform incr [1, 2, 3, 4]
λ> transform incr (Branch 1 (Leaf 2) (Leaf 3))
λ> transform incr (Queue [1, 2, 3, 4])
```

## Terus Functor itu Apa?

Daritadi ngalor ngidul bahas "transformasi" tapi gak pernah nyinggung sama sekali tentang Functor. Eits jangan salah, sedari awal kita sebenarnya sudah bahas functor: Functor adalah class Transformable itu sendiri! Bedanya method `transform` dinamai `map`.

```hs
class Functor f
  map :: ∀ a b. (a -> b) -> f a -> f b
```

### Laws

Jadi, Functor adalah sebuah type class dengan `f :: Type -> Type` yang memiliki method tunggal `map`. Plus, ada dua "hukum" yang harus dipatuhi agar benar dikatakan functor:

1. Identity Law. `map identity f == f`
2. Composition Law. `map x (map y f) == map (x <<< y) f`

Identity Law memastikan kesamaan struktur data setelah transformasi, walaupun value di dalamnya bisa jadi berbeda. Jangan terlalu ambil pusing sama hukum-hukum ini. Cukup tau aja 😄

### Kind

Bagaimana dengan tipe data yang memiliki kind lebih dari `Type -> Type` seperti Tuple atau Either? Apa masih bisa punya instance Functor?

```hs
data Tuple a b = Tuple a b
data Either a b = Left a | Right b

λ> :k Tuple
Type -> Type -> Type
λ> :k Either
Type -> Type -> Type
```

Yap. Masih bisa dong. Apply secara partial saja, kan type constructor juga auto-curry di Purescript 😉 Sebagai konsekuensinya, hanya sisa type variable **(yang paling kanan) saja yang bisa ditransformasi**.

```hs
instance functorTuple :: Functor (Tuple a) where
  map fn (Tuple a b) = Tuple a (fn b) -- `a` is unchanged

instance functorEither :: Functor (Either a) where
  map _  (Left a)  = Left a -- unchanged
  map fn (Right b) = Right (fn b)
```

### Penerapan

Penerapan Functor sangat sangat banyak. Sudah dimana-mana. Artikel ini bakal jadi puanjang kalau dibahas satu-satu 😅. Alih-alih saya buatkan list saja dan temen-temen bisa langsung intip ke TKP:

1. [Array](https://github.com/purescript/purescript-prelude/blob/a96663b34364fdd0885a200955e35b99f4e58c43/src/Data/Functor.purs#L42-L43)
2. [Maybe](https://github.com/purescript/purescript-maybe/blob/81f0397636bcbca28642f62421aebfd9e1afa7fb/src/Data/Maybe.purs#L32-L34)
3. [Either](https://github.com/purescript/purescript-either/blob/8b4b38a729f8e88750b03e5c7baf2b3863ce4742/src/Data/Either.purs#L38)
4. [Tuple](https://github.com/purescript/purescript-tuples/blob/0036bf9d99b721fd0f2e539d24e18e484b016927/src/Data/Tuple.purs#L93-L99)
5. [Function!!](https://github.com/purescript/purescript-prelude/blob/a96663b34364fdd0885a200955e35b99f4e58c43/src/Data/Functor.purs#L39-L40)
6. [Tree](https://github.com/jacobstanley/purescript-jack/blob/d47ab4f9d321a299f34ec16a98ad787585e610f9/src/Jack/Tree.purs#L51-L53)
7. [Queue](https://github.com/purescript/purescript-catenable-lists/blob/d81b7df30d9879d0bb531b3102fb36f429c2f12e/src/Data/CatQueue.purs#L172-L173)
8. [NonEmptyList](https://github.com/purescript/purescript-lists/blob/6629e0c05f2ee4b47a9b2fbcdbe6619ff17e8e28/src/Data/List/Types.purs#L177)
9. [Parser](https://github.com/purescript-contrib/purescript-parsing/blob/e801a0ef42f3211b1602a94a269eef7ce551423f/src/Text/Parsing/Parser.purs#L93)
10. endless possibilities..

## Penutup

Saya kira sampai di sini dulu penjelasan tentang Functor. Saya harap kita belajar banyak dari Functor soal bagaimana HOC dan generic berperan penting dalam code reusability.

Ngomong-ngomong saya sedang menulis artikel lain tentang pembahasan apakah semua struktur data dengan kind `Type -> Type` bisa otomatis menjadi Functor. Insight ini menarik karena dengan menganalisanya, kita bisa menemukan motivasi di balik Contravariant Functor, saudara tiri Functor. Insyallah rampung dalam waktu dekat.

Semoga series FP ini bermanfaat. Salam 🙂


