---
title: "Bifunctor: Sikat Kanan Kiri"
date: 2020-01-23T10:02:51+01:00
description: "Functor + Functor = Bifunctor"
images: ["/uploads/dual-trees.jpg"]
image:
  src: "/uploads/dual-trees.jpg"
  caption: Image by <a href="https://pixabay.com/users/skeeze-272447/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2242958">skeeze</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2242958">Pixabay</a>
tags: ["purescript", "haskell", "functor", "functionalprogramming"]
categories: ["programming", "purescript", "functional programming"]
draft: false
---

[Functor]({{< ref "./code-reuse-berkaca-dari-functor.md" >}}) adalah sebuah struktur data yang membungkus value lain, yang strukturnya terjaga (_preserved_) saat transformasi (`map`). Functor hanya dapat membungkus satu value saja, dan hanya satu value ini saja yang dapat diubah ke bentuk lain.

```hs
data Tuple a b = Tuple a b

instance functorTuple :: Functor (Tuple a) where
  map fn (Tuple a b) = Tuple a (fn b) -- `a` is unchanged

λ> map (_ + 1) (Tuple 5 8)
Tuple 5 9
```

Di contoh ini Tuple adalah struktur data yang memiliki dua buah value, `a` dan `b`. Namun karena sifat Functor yang hanya bisa mentransformasi satu value saja, kita harus korbankan `a` dan kasih jalan ke `b`.

Lalu bagaimana jika kita ingin mentransformasi **kedua value** Tuple?

Pake <strong style="font-size: 2.5rem; color: #e81111">BIFUNCTOR</strong> bro!!

Set ngegas.

---

Oke, alih-alih menerima satu function transformasi saja seperti Functor, Bifunctor menerima **dua** buah function transformasi sekaligus: yang satu untuk mengubah nilai `a` dan yang satunya lagi untuk mengubah nilai `b`.

```hs
λ> bimap (_ - 1) (_ + 1) (Tuple 5 8)
Tuple 4 9
```

Dengan ini kedua buah value akhirnya bisa ditransformasi.

Contoh lain. `Either`.

```hs
data Either a b = Left a | Right b

λ> bimap toUpper (_ + 1) (Left "error!")
Left "ERROR!"
λ> bimap toUpper (_ + 1) (Right 8)
Right 9
```

Pola ini sangat mudah dicerna sehingga bisa kita buatkan typeclass-nya sendiri, yang akan kita namakan `Bifunctor`, yang memiliki method bernama `bimap`, yang mengambil dua buah function, yang tetap _preserve_ struktur data tersebut, yang yaang bikinin teh dong.

```hs
class Bifunctor f where
  bimap :: forall a b c d. (a -> b) -> (c -> d) -> f a c -> f b d
```

Karena `bimap` adalah method dari sebuah typeclass, seseorang gak bisa begitu saja menggunakan method ini. Struktur data yang dimanipulasi harus terlebih dahulu memiliki instance Bifunctor.

```hs
instance bifunctorTuple :: Bifunctor Tuple where
  bimap f g (Tuple x y) = Tuple (f x) (g y)

instance bifunctorEither :: Bifunctor Either where
  bimap f _ (Left l) = Left (f l)
  bimap _ g (Right r) = Right (g r)
```

Semua struktur data yang memiliki dua buah type paramater dapat dijadikan Bifunctor selama kedua type parameter-nya covariant. Maaf, ini kalimat sebenarnya kopas aja dari [sini](https://github.com/purescript/purescript-bifunctors/blob/1062425892b4a1c734ec653dded22546e3063b27/src/Data/Bifunctor.purs#L7-L8). Bisa di-skip karena gak peting. Tapi kalau penasaran, saya include gist dari apa yang dimaksud dengan Covariant dan Contravariant 👇🏻

{{< figure src="/uploads/covariant-contravariant.png" alt="Rumus Covariant & Contravariant" caption="Rumus Covariant & Contravariant ([sumber](https://www.youtube.com/watch?v=OJtGECfksds&t=1142s))" class="fig-center" >}}

Sekian dan terima kangen.
