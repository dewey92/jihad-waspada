---
title: "Generic di atas Generic: Higher-Kinded Type"
date: 2019-07-06T15:30:24+02:00
description: "Setiap value ada type-nya. Dan setiap type ada kind-nya."
images: ["/uploads/stairs.jpg"]
image:
  src: "/uploads/stairs.jpg"
  caption: Photo by <strong><a href="https://www.pexels.com/@blitzboy?utm_content=attributionCopyText&amp;utm_medium=referral&amp;utm_source=pexels">Sindre Strøm</a></strong> from <strong><a href="https://www.pexels.com/photo/photo-of-sneakers-1497406/?utm_content=attributionCopyText&amp;utm_medium=referral&amp;utm_source=pexels">Pexels</a></strong>
tags: ["typescript", "haskell", "purescript", "polymorphism", "types"]
categories: ["programming", "type system", "typescript", "purescript"]
draft: false
---

Artikel ini merupakan artikel lanjutan dari [materi Generics dari Medium saya](https://medium.com/codewey/kuy-ngobrol-dikit-soal-generics-4a05f60d5be1) menggunakan Typescript.

## Generics
### Array
Sebelum membahas lebih dalam tentang Higher-Kinded Type, ada baiknya kita mengingat-ingat kembali apa itu Generics. Dan saya rasa gak ada data structure generic yang lebih sederhana dan umum digunakan di Typescript kecuali Array.

```ts
interface Array<T> {
  // ... map, filter, reduce, length, dll
}

const languages: Array<string> = ['javascript', 'typescript', 'purescript']
```

Karena struktur data ini generic, kita bisa membuat function yang generic pula untuk mengubah nilai yang ada di dalamnya tanpa memandang apakah ia bertipe `number`, `string`, `object`, atau yang lainnya. _Let's say_, kita lagi butuh function yang bisa **mengubah array menjadi array of tuple**.

```ts {hl_lines=[2]}
const tuplifyArray = <T>(array: Array<T>): Array<[T, T]> =>
  array.map(item => [item, item])

const int = tuplifyArray([1, 2, 3])
// [[1, 1], [2, 2], [3, 3]]

const str = tuplifyArray(['tikus', 'makan', 'sabun'])
// [['tikus', 'tikus'], ['makan', 'makan'], ['sabun', 'sabun']]
```

So, function `tuplifyArray` ini bersifat _polymorphic_: dimana ia mengubah `Array<number>` menjadi `Array<[number, number]>` pada contoh pertama dan mengubah `Array<string>` menjadi `Array<[string, string]>` pada contoh kedua, gak pandang type di dalamnya 🔥 So far so good.

### Tree
Beberapa bulan kemudian, Project Manager [dateng bawa berita duka](https://www.instagram.com/p/Bk4m_FrB0pR/): kita diminta untuk menambahkan struktur data Tree karena ternyata _requirement_ proyek sudah mulai melebar.

```ts
type Tree<T> = Leaf<T> | Branch<T>
type Leaf<T> = {
  _type: 'leaf';
  value: T;
}
type Branch<T> = {
  _type: 'branch';
  left: Tree<T>;
  right: Tree<T>;
}

// -- helper functions --
const leaf = <T>(value: T): Leaf<T> => ({
  _type: 'leaf',
  value,
})
const branch = <T>(left: Tree<T>, right: Tree<T>): Branch<T> => ({
  _type: 'branch',
  left,
  right,
})

// -- usage --
const treeA: Tree<number> = branch(
  leaf(1),
  branch(
    leaf(2),
    leaf(3)
  )
)
```

Dasar emang PM ini plin-plan dan kurang bisa nego sama client, dia sekarang minta juga untuk dibuatkan fungsi `tuplifyTree` persis seperti fungsi `tuplifyArray` yang sudah dibuat tadi.

```ts {hl_lines=[4,16,"18-19"]}
function tuplifyTree<T>(tree: Tree<T>): Tree<[T, T]> {
  switch (tree._type) {
    case 'leaf':
      return leaf([tree.value, tree.value])
    case 'branch':
      return branch(
        tuplifyTree(tree.left),
        tuplifyTree(tree.right)
      )
  }
}

const result = tuplifyTree(treeA)
/* result:
branch(
  leaf([1, 1]),
  branch(
    leaf([2, 2]),
    leaf([3, 3])
  )
)
*/
```

Kejadian ini membuat kita berasumsi jauh ke depan bagaimana jika suatu saat nanti lingkup projek menjadi semakin lebar dan harus menghadirkan beberapa data structure generic baru beserta function `tuplify`-nya. Data structure tersebut bisa berupa `Stack<T>`, `Queue<T>`, `LinkedList<T>`, `Stream<T>`, _you name it_.

_And this is the time_: untuk membuat abstraksi dari solusi yang diinginkan, sebagai seorang developer kita harus bisa melihat pola dari masalah yang ada terlebih dahulu. Mari kita bandingkan function type `tuplifyArray` dengan `tuplifyTree`, maka kita akan menemukan bahwa kedua function type tersebut ternyata memiliki pola yang sangat mirip:

```ts
tuplifyArray<T>(array: Array<T>): Array<[T, T]>
tuplifyTree <T>( tree:  Tree<T>):  Tree<[T, T]>
```

Yang membedakan keduanya hanyalah type `Array` dan `Tree`. AHA-moment datang 💡, sekarang kita mulai membayangkan solusi yang lebih generic, yang mengasbtraksi (tidak lagi terikat dengan) `Array` maupun `Tree`:

```ts
tuplify<F, T>(wrapper: F<T>): F<[T, T]>

tuplify(array) // F = Array
tuplify(tree)  // F = Tree
```

Namun sayangnya, compiler Typescript menolak syntax ini dengan pesan error: `Type 'F' is not generic`. Error muncul karena keterbatasan compiler Typescript yang tidak memperbolehkan sebuah type variable menerima type lain.

Nah kemampuan type variable menerima type lain (disebut type constructor) inilah yang disebut Higher-Kinded Type. Namun sayangnya Typescript hanya bisa mengabstraksi type, belum bisa mengabstraksi type constructor.

## Kind, type of type

Kind adalah cara untuk mengekspresikan (concrete) type dan type constructor dengan symbol `*`. Anggap saja Kind ini adalah type-nya type 😄

`*` adalah kind untuk concrete types. `string`, `number`, `Date`, `Array<number>`, `Tree<string>` masuk dalam kategori ini.

`* -> *` adalah kind untuk type constructor yang menerima satu type parameter. Kita bisa membacanya dengan "sesuatu yang menerima `*` (concrete type) dan mengembalikan `*` (concrete type pula)". Contohnya adalah `Array`, dan `Tree` di atas.

`* -> * -> *` adalah kind untuk type constructor dengan dua type parameter. `Record` salah satu contohnya, karena kita harus menyuplai dua type paramater untuk menghasilkan sebuah concrete type: `Record<string, number>`.

`* -> * -> * -> *` adalah kind untuk type constructor dengan tiga type parameter, dsb.

Di salah satu bahasa yang mendukung HKT seperti Purescript (atau Haskell), permasalah `tuplify` dapat diekspresikan dengan:

```purs {hl_lines=[1,2]}
class Tupleable f where
  tuplify :: f t -> f (t, t)

instance Tupleable Array where
  tuplify = ... -- implementasi tuplify Array

instance Tupleable Tree where
  tuplify = ... -- implementasi tuplify Tree
```

Type parameter `f` bisa disubstitusi dengan `Array` dan `Tree`, artinya `f` memiliki Kind `* -> *`. Dan type paramater `t`-nya adalah concrete type (Kind `*`). Anggap aja `class` di atas itu seperti `interface` di Typescript dan `instance` itu seperti `class` :p

```ts {hl_lines=[2,14,15],linenos=inline}
interface Tupleable<T> {
  tuplify(): Tupleable<[T, T]>
}

class Array<T> implements Tupleable<T> {
  constructor(private val: T) {}
  tuplify(): Array<[T, T]> {
    return new Array([this.val]);
  }
}

class Tree<T> implements Tupleable<T> {
  constructor(private val: T) {}
  tuplify(): Array<[T, T]> {
    return new Array([this.val]);
  }
}
```

Namun tetap masih kurang akurat: perhatikan return type baris 14-15. Return type dari fungsi `tuplify` **bisa berupa apa saja asalkan ia subtype dari `Tupleable`**. Artinya notasi `Tree<T> → Array<[T, T]>` di baris 14 type-checked! Sedangkan kita ingin menjaga type constructor yang sama di return type: `Array<T> → Array<[T, T]>` atau `Tree<T> → Tree<[T, T]>` saja.

Maunya sih begini tapi gak bisa 😅

```ts
interface Tupleable<T> {
  tuplify(): this<[T, T]>
}
```

Selain itu, konsep Higher-Kinded Type memungkinkan kita untuk membuat _factory type_, yang berarti satu type dapat menghasilkan banyak type atau data structure.

```purs
-- kind `User` adalah `(* -> *) -> *`
type User box = {
  name :: String,
  email :: box String
}

type Identity a = a

type UserWithEmail = User Identity
type UserWithOrWithoutEmail = User Maybe
type UserWithManyEmails = User Array
...
-- endless possibilities
```

## Penutup

Ada beberapa library yang mencoba mensimulasikan konsep HKT ke dalam Typescript. Yang paling terkenal adalah [fp-ts](https://github.com/gcanti/fp-ts). Dulu kami pernah menggunakan library ini di production, tapi kami merasa terlalu verbose dan kurang cocok dengan tim sehingga penggunaannya kami minimalisir. Ada juga alternatif lain yang mungkin lebih simple, [hkts](https://github.com/pelotom/hkts), belum pernah nyoba tapi hehe.

Akhirul kalam, semoga artikel ini dapat menambah wawasan temen-temen soal type. Kalau mau kasih komentar atau masukan, bisa langsung ke thread Twitter saya aja biar diskusinya asik 😉.

{{< tweet "1147535086453178369" >}}

<br />

_Arriverdeci!_
