---
title: "Higher Kinded Type"
date: 2019-07-04T12:57:24+02:00
description: ""
images: [""]
image:
  src: ""
  caption: ""
tags: [""]
categories: [""]
draft: true
---

Artikel ini merupakan artikel lanjutan dari [materi Generics yang awalnya ditulis di Medium saya](https://medium.com/codewey/kuy-ngobrol-dikit-soal-generics-4a05f60d5be1). Cuman ya karena [saya udah gak nulis di Medium lagi]({{< ref "/post/selamat-tinggal-medium.md" >}}), jadinya saya lanjutin di sini aja hehe.

> PS: Saya bukan orang yang ahli dalam bidang Type System atau Programming Language, cuman seneng main sama statically-typed language dan fitur-fiturnya.

Kalau temen-temen pernah baca-baca atau belajar Functional Programming dengan bahasa yang **strongly static typed**, mungkin di saat-saat tertentu pernah mendengar istilah _Higher Kinded Type_ atau disingkat HKT. Konsep ini lumayan _mindblowing_ 🤯 (_at least_ buat saya), karena memungkinkan kita untuk **membuat abstraksi** yang gak akan ditemui di bahasa-bahasa mainstream seperti Typescript, Flow, Java, atau C#. Konsep HKT digunakan di beberapa bahasa pemrograman seperti Haskell, Purescript, dan Scala yang mana ketiganya memiliki konsep [klasifikasi types atau lumrah disebut Typeclasses](https://en.wikipedia.org/wiki/Type_class). Tapi kita gak akan bahas Typeclasses sekarang. ~~Kita bahas pernikahan dan tips-tips membangun rumah tangga aja.~~ Yang akan menjadi titik bahasan dalam artikel ini adalah menjelaskan konsep HKT menggunakan Typescript.

# Generics
## Array
Sebelum membahas lebih dalam tentang Higher-Kinded Type, ada baiknya kita mengingat-ingat kembali apa itu Generics. Dan, gak ada _data structure_ Generics yang lebih sederhana dan umum digunakan di dunia Typescript kecuali Array.

{{< highlight typescript >}}
interface Array<T> {
  // ... map, filter, reduce, length, dll
}

const languages: Array<string> = ['javascript', 'typescript', 'purescript']
{{< /highlight >}}

> <a name="intermezzo"></a> ⚠️ IMPORTANT: `string`, `number`, `Date`, dll **yang tidak menerima generic disebut "type"**. Sedangkan `Array` **disebut "type constructor" karena dapat menghasilkan type (dengan mengambil generic)**. Bagaimana dengan `Array<string>`? Apakah termasuk "type" atau "type constructor"? Jawabannya: "type".

Karena struktur data ini generics, kita bisa membuat function yang generics pula untuk mengubah nilai yang ada di dalamnya tanpa memandang apakah ia bertipe `number`, `string`, `object`, atau yang lainnya. _Let's say_, kita lagi butuh banget function yang bisa **ngubah array menjadi array of array**. Kalau gak ada function ini, proyek mangkrak ~~dan lu bakal dipaksa kawin ama anak kampung sebelah~~.

{{< highlight typescript "hl_lines=2,noclasses=false" >}}
const arrayifyArray = <A>(array: Array<A>): Array<[A]> =>
  array.map(item => [item])

const result = arrayifyArray([1, 2, 3])
// [[1], [2], [3]]

const result2 = arrayifyArray(['tikus', 'makan', 'sabun'])
// [['tikus'], ['makan'], ['sabun']]
{{< /highlight >}}

So, function `arrayifyArray` ini bersifat _polymorphic_: dimana ia mengubah `Array<number>` menjadi `Array<[number]>` pada contoh pertama dan mengubah `Array<string>` menjadi `Array<[string]>` pada contoh kedua 🔥 So far so good.

## Tree
Beberapa bulan kemudian, Project Manager [dateng bawa berita duka](https://www.instagram.com/p/Bk4m_FrB0pR/): kita diminta untuk menambahkan Tree _data structure_ (seperti yang ada [di artikel bagian 1](https://medium.com/codewey/kuy-ngobrol-dikit-soal-generics-4a05f60d5be1)) karena ternyata _requirement_ proyek mulai melebar.

{{< highlight typescript >}}
type Tree<A> = Leaf<A> | Branch<A>
type Leaf<A> = {
  _type: 'leaf';
  value: A;
}
type Branch<A> = {
  _type: 'branch';
  left: Tree<A>;
  right: Tree<A>;
}

// -- helper functions --
const leaf = <A>(value: A): Leaf<A> => ({
  _type: 'leaf',
  value,
})
const branch = <A>(left: Tree<A>, right: Tree<A>): Branch<A> => ({
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

Dasar emang PM ini plin-plan dan gak bisa nego sama client, dia sekarang minta juga untuk dibuatkan fungsi `arrayifyTree` persis seperti fungsi `arrayifyArray` yang sudah dibuat tadi. Sebagai developer [bike-bike](https://www.instagram.com/p/BvOSn0pD0A5/), kita buatkan saja fungsi tersebut:

{{< highlight typescript "hl_lines=4 16 18-19,noclasses=false" >}}
function arrayifyTree<A>(tree: Tree<A>): Tree<[A]> {
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

Melihat tabiat PM yang agak menjengkelkan ini, kita mulai membuat asumsi-asumsi _worst case_ dimana bisa jadi suatu saat nanti PM akan datang kembali dengan _data structures_ Generics yang baru beserta function `arrayify`-nya: _data structures_ tersebut bisa berupa `Stack<A>`, `Queue<A>`, `LinkedList<A>`, `Stream<A>`, atau bahkan `Maybe<A>`.

_This is the time_: untuk membuat abstraksi dari solusi yang diinginkan, seorang developer harus bisa melihat pola dari masalah yang ada terlebih dahulu. Jika function `arrayifyArray` dan `arrayifyTree` dibandingkan, kita akan melihat bahwa kedua function tersebut ternyata memiliki pola yang sangat mirip:

```hs
arrayifyArray<A>(array: Array<A>): Array<[A]>
arrayifyTree <A>( tree:  Tree<A>):  Tree<[A]>
```

Yang membedakan keduanya hanyalah `Array` dan `Tree`. Dari pola ini, sekarang kita mulai membayangkan **solusi yang lebih generic yang tidak lagi terikat dengan "type constructor" `Array` maupun `Tree`**:

```hs
arrayify<DS, A>(dataStructure: DS<A>): DS<[A]>
```

Namun sayangnya, compiler Typescript menolak syntax ini dengan error message: `Type 'DS' is not generic`. Ini adalah keterbatasan compiler Typescript yang menganggap **semua type variable yang muncul (termasuk `DS`) haruslah berupa "type", tidak boleh berupa "type constructor"** <sup>[[1](#intermezzo)]</sup>.

Workaround-nya adalah dengan menjadikan `DS` sebuah interface agar nantinya dapat disubstitusi dengan `Array` maupun `Tree` (bisa juga `Stack`, `Queue`, dll).

{{< highlight typescript "hl_lines=1,noclasses=false" >}}
interface DS<A> {}

interface Array<A> extends DS<A> {}
interface Tree<A> extends DS<A> {}
interface Stack<A> extends DS<A> {}
interface Queue<A> extends DS<A> {}
// dsb ...

function arrayify<A>(dataStructure: DS<A>): DS<[A]> { ... }
{{< /highlight >}}

🎉🎉🎉 Type-checked!

Tapi tunggu dulu, workaround di atas masih bisa dibilang _leaky_ jika kita perhatikan return type-nya. Return type dari fungsi `arrayify` **bisa berupa apa saja asal merupakan subtype dari `DS`**. Artinya notasi `arrayify<A>(ds: Array<A>): Tree<[A]>` type-checked! Sedangkan yang kita inginkan adalah input dan output memiliki tipe yang sama: antara `arrayify<A>(ds: Array<A>): Array<[A]>` atau `arrayify<A>(ds: Tree<A>): Tree<[A]>`. Adakah cara yang lebih baik?

Jawaban singkatnya: nggak di Typescript. Silakan tutup artikel. Tidur 😃

Jawaban panjangnya: silakan dilanjutkan 👇🏻

# Kinds
