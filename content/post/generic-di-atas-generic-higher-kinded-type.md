---
title: "Generic di atas Generic: Higher Kinded Type"
date: 2019-07-06T15:30:24+02:00
description: "Setiap value ada type-nya. Dan setiap type ada kind-nya."
images: ["/uploads/stairs.jpg"]
image:
  src: "/uploads/stairs.jpg"
  caption: Photo by <strong><a href="https://www.pexels.com/@blitzboy?utm_content=attributionCopyText&amp;utm_medium=referral&amp;utm_source=pexels">Sindre Strøm</a></strong> from <strong><a href="https://www.pexels.com/photo/photo-of-sneakers-1497406/?utm_content=attributionCopyText&amp;utm_medium=referral&amp;utm_source=pexels">Pexels</a></strong>
tags: ["typescript", "haskell", "purescript"]
categories: [""]
draft: false
---

Artikel ini merupakan artikel lanjutan dari [materi Generics yang awalnya ditulis di Medium saya](https://medium.com/codewey/kuy-ngobrol-dikit-soal-generics-4a05f60d5be1). Cuman ya karena [saya udah gak nulis di Medium lagi]({{< ref "/post/selamat-tinggal-medium.md" >}}), jadinya saya lanjutin di sini aja hehe.

_PS: Saya bukan orang yang ahli dalam bidang Type Theory atau Programming Language, cuman seneng main sama statically-typed language sekaligus ingin mendalami bagaimana types dapat membantu dalam pengembangan software yang robust dan expressive_

Kalau temen-temen pernah baca-baca atau belajar Functional Programming menggunakan bahasa dengan static typing yang setrong, mungkin di saat-saat tertentu pernah mendengar istilah Higher Kinded Type atau disingkat HKT. Konsep ini lumayan _mindblowing_ 🤯 (_at least_ buat saya), karena memungkinkan kita untuk **membuat abstraksi** yang gak akan ditemui di bahasa-bahasa mainstream seperti Typescript, Flow, Java, atau C#. Konsep HKT digunakan di beberapa bahasa pemrograman seperti Haskell, Purescript, dan Scala yang mana ketiganya memiliki konsep [klasifikasi types atau lumrah disebut Typeclasses](https://en.wikipedia.org/wiki/Type_class). Tapi kita gak akan bahas Typeclasses sekarang. ~~Kita bahas pernikahan dan tips-tips membangun rumah tangga aja.~~ Yang akan menjadi titik bahasan dalam artikel ini adalah menjelaskan konsep HKT menggunakan Typescript.

# Generics
## Array
Sebelum membahas lebih dalam tentang Higher-Kinded Type, ada baiknya kita mengingat-ingat kembali apa itu Generics. Dan, gak ada data structure generic yang lebih sederhana dan umum digunakan di dunia Typescript kecuali Array.

{{< highlight typescript >}}
interface Array<T> {
  // ... map, filter, reduce, length, dll
}

const languages: Array<string> = ['javascript', 'typescript', 'purescript']
{{< /highlight >}}

> ⚠️ IMPORTANT: type seperti `string`, `number`, `Date`, dll **yang tidak menerima generic disebut "(concrete) type"**. Sedangkan `Array` **disebut "type constructor" karena dapat menghasilkan concrete type (dengan mengambil generic)**. Bagaimana dengan `Array<string>`? Apakah termasuk "type" atau "type constructor"? Jawabannya: "type".

Karena struktur data ini generic, kita bisa membuat function yang generic pula untuk mengubah nilai yang ada di dalamnya tanpa memandang apakah ia bertipe `number`, `string`, `object`, atau yang lainnya. _Let's say_, kita lagi butuh banget function yang bisa **ngubah array menjadi array of array**. Kalau gak ada function ini, proyek mangkrak ~~dan lu bakal dipaksa kawin ama anak kampung sebelah~~.

{{< highlight typescript "hl_lines=2,noclasses=false" >}}
const arrayifyArray = <T>(array: Array<T>): Array<[T]> =>
  array.map(item => [item])

const result = arrayifyArray([1, 2, 3])
// [[1], [2], [3]]

const result2 = arrayifyArray(['tikus', 'makan', 'sabun'])
// [['tikus'], ['makan'], ['sabun']]
{{< /highlight >}}

So, function `arrayifyArray` ini bersifat _polymorphic_: dimana ia mengubah `Array<number>` menjadi `Array<[number]>` pada contoh pertama dan mengubah `Array<string>` menjadi `Array<[string]>` pada contoh kedua, gak pandang type di dalamnya 🔥 So far so good.

## Tree
Beberapa bulan kemudian, Project Manager [dateng bawa berita duka](https://www.instagram.com/p/Bk4m_FrB0pR/): kita diminta untuk menambahkan struktur data Tree karena ternyata _requirement_ proyek sudah mulai melebar.

{{< highlight typescript >}}
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
{{< /highlight >}}

Dasar emang PM ini plin-plan dan kurang bisa nego sama client, dia sekarang minta juga untuk dibuatkan fungsi `arrayifyTree` persis seperti fungsi `arrayifyArray` yang sudah dibuat tadi. Sebagai developer [bike-bike](https://www.instagram.com/p/BvOSn0pD0A5/), kita buatkan saja fungsinya:

{{< highlight typescript "hl_lines=4 16 18-19,noclasses=false" >}}
function arrayifyTree<T>(tree: Tree<T>): Tree<[T]> {
  switch (tree._type) {
    case 'leaf':
      return leaf([tree.value])
    case 'branch':
      return branch(
        arrayifyTree(tree.left),
        arrayifyTree(tree.right)
      )
  }
}

const result = arrayifyTree(treeA)
/* result:
branch(
  leaf([1]),
  branch(
    leaf([2]),
    leaf([3])
  )
)
*/
{{< /highlight >}}

Kejadian ini membuat kita berasumsi jauh ke depan bagaimana jika suatu saat nanti lingkup projek menjadi semakin lebar dan harus menghadirkan beberapa data structure generic baru beserta function `arrayify`-nya. Data structure tersebut bisa berupa `Stack<T>`, `Queue<T>`, `LinkedList<T>`, `Stream<T>`, _you name it_.

_And this is the time_: untuk membuat abstraksi dari solusi yang diinginkan, sebagai seorang developer kita harus bisa melihat pola dari masalah yang ada terlebih dahulu. Mari kita bandingkan function type `arrayifyArray` dengan `arrayifyTree`, maka kita akan menemukan bahwa kedua function type tersebut ternyata memiliki pola yang sangat mirip:

```hs
arrayifyArray<T>(array: Array<T>): Array<[T]>
arrayifyTree <T>( tree:  Tree<T>):  Tree<[T]>
```

Yang membedakan keduanya hanyalah type constructor `Array` dan `Tree`. AHA-moment datang 💡, sekarang kita mulai membayangkan solusi yang lebih generic, yang mengasbtraksi (tidak lagi terikat dengan) "type constructor" `Array` maupun `Tree`:

```hs
arrayify<DS, T>(arrayable: DS<T>): DS<[T]>
```

Cerdas dah kite 🥳🎉🎊

Namun sayangnya, compiler Typescript menolak syntax ini dengan pesan error: `Type 'DS' is not generic`. Error muncul karena keterbatasan compiler Typescript yang mengharuskan **semua type variable (termasuk `DS`) berupa "type", tidak boleh berupa "type constructor"**. Sedangkan dalam kasus ini kita menginginkan type variable `DS` berupa type constructor.

_Anyway_, **kemampuan mengabstraksi type constructor inilah yang disebut Higher-Kinded Type**. Namun sayangnya Typescript hanya bisa mengabstraksi type, belum bisa mengabstraksi type constructor. Ya sudah, tidur sanah, aku kecewa :')

# Kind, type of type
Udah tidurnya bro? Hehe. Oke kita lanjut. Kind adalah cara untuk mengekspresikan (concrete) type dan type constructor dengan symbol `*`. Anggap saja Kind ini adalah type-nya type 😄

`*` adalah kind untuk concrete types. `string`, `number`, `Date`, `Array<number>`, `Tree<string>` masuk dalam kategori ini.

`* -> *` adalah kind untuk type constructor yang menerima satu type parameter. Kita bisa membacanya dengan "sesuatu yang menerima `*` (concrete type) dan mengembalikan `*` (concrete type pula)". Contohnya adalah `Array`, dan `Tree` di atas.

`* -> * -> *` adalah kind untuk type constructor dengan dua type parameter. `Record` salah satu contohnya, karena kita harus menyuplai dua type paramater untuk menghasilkan sebuah concrete type: `Record<string, number>`.

`* -> * -> * -> *` adalah kind untuk type constructor dengan tiga type parameter, dsb.

Balik ke pembahasan Higher-Kinded Type. Di Haskell sendiri &mdash; salah satu bahasa dengan type system paling canggih &mdash; permasalah `arrayify` dapat dituliskan dengan HKT.

{{< highlight hs "noclasses=false,hl_lines= 1 2" >}}
class Arrayable a where
  arrayify :: a t -> a [t]

instance Arrayable Array where
  arrayify = ... -- implementasi arrayify Array

instance Arrayable Tree where
  arrayify = ... -- implementasi arrayify Tree
{{< /highlight >}}

Type parameter `a` bisa disubstitusi dengan `Array` dan `Tree`, artinya `a` memiliki Kind `* -> *`. Dan type paramater `t`-nya adalah concrete type (Kind `*`). Anggap aja `class` di atas itu seperti `interface` di Typescript dan `instance` itu seperti `class` :p

Lagi-lagi, kalau "style" di atas diterapkan di Typescript, saya rasa tetap belum bisa sepenuhnya akurat.

{{< highlight typescript "hl_lines=2 14 15,linenos=inline" >}}
interface Arrayable<T> {
  arrayify(): Arrayable<[T]>
}

class Array<T> implements Arrayable<T> {
  constructor(private val: T) {}
  arrayify(): Array<[T]> {
    return new Array([this.val]);
  }
}

class Tree<T> implements Arrayable<T> {
  constructor(private val: T) {}
  arrayify(): Array<[T]> {
    return new Array([this.val]);
  }
}
{{< /highlight >}}

Kurang akuratnya solusi ini terlihat jika kita memperhatikan return type-nya. Return type dari fungsi `arrayify` **bisa berupa apa saja asalkan ia subtype dari `Arrayable`**. Artinya notasi `Tree<T> → Array<[T]>` seperti yang ada pada line 14 type-checked! Sedangkan yang kita inginkan adalah kesamaan type constructor (dalam hal ini ya class itu sendiri) dengan outputnya: `Array<T> → Array<[T]>` atau `Tree<T> → Tree<[T]>` saja.

Maunya sih begini tapi gak bisa 😅

{{< highlight typescript >}}
interface Arrayable<T> {
  arrayify(): this<[T]>
}
{{< /highlight >}}

Selain itu, konsep Higher-Kinded Type memungkinkan kita untuk membuat _factory ~~function~~ type_, yang berarti satu type dapat menghasilkan banyak type atau data structure.

```hs
-- kind `User` adalah `(* -> *) -> *`
type User box = {
  name :: String,
  email :: box String
}

type Id a = a

type UserWithEmail = User Id
type UserWithOrWithoutEmail = User Maybe
type UserWithManyEmails = User Array
...
-- endless possibilities
```

Keren yak 🎉

# Penutup

Jika teman-teman tertarik belajar Haskell atau Purescript yang memiliki konsep Typeclasses, memahami Higher-Kinded Type dapat membantu intuisi kita dalam mencerna suatu data structure dan/atau type. Saya sendiri juga masih belajar Haskell atau Purescript pelan-pelan, dan setiap kali melihat solusi-solusi di internet yang menggunakan HKT, saya cuman bisa bergumam dalam hati: "Andaikata Typescript sudah support HKT ...".

Beberapa library mencoba meng-encode konsep HKT ke dalam Typescript. Yang paling terkenal adalah [fp-ts](https://github.com/gcanti/fp-ts). Dulu kami pernah menggunakan library ini di production, tapi kami merasa terlalu verbose dan kurang cocok dengan tim sehingga penggunaannya kami minimalisir. Ada juga alternatif lain yang mungkin lebih simple, [hkts](https://github.com/pelotom/hkts), belum pernah nyoba tapi hehe.

Akhirul kalam, semoga artikel ini dapat menambah wawasan temen-temen dalam dunia programming terutama di bidang type theory. Kalau mau ngasih komentar atau masukan, [bisa langsung ke Twitter saya aja, kita diskusi disitu biar asik](https://twitter.com/Dewey92/status/1147535086453178369) 😉. _Arriverdeci!_
