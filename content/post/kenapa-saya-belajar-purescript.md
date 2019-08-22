---
title: "Kenapa Saya Belajar Purescript"
date: 2019-08-03T17:22:55+02:00
description: "Sekedar share pendapat pribadi kenapa lebih memilih Purescript dibandingkan bahasa-bahasa functional lainnya"
images: ["/uploads/question-marks.jpg"]
image:
  src: "/uploads/question-marks.jpg"
  caption: Image by <a href="https://pixabay.com/users/qimono-1962238/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2492009">Arek Socha</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2492009">Pixabay</a>
tags: ["purescript", "functionalprogramming", "types"]
categories: ["programming", "purescript"]
draft: false
---

Singkat cerita: karena mau belajar _typed functional programming_.

## Kenapa Harus _Typed_?
Setelah hampir dua tahun semenjak pertama kali menggunakan Typescript, saya menyimpulkan bagaimana pentingnya static typing dalam membuat program, terutama program dalam skala yang tidak kecil. Selain dapat membantu produktifitas programmer, types juga dapat membuat code menjadi ekspresif dan relatif lebih aman saat runtime. Ekspresif dalam arti tidak perlu melakukan banyak pengecekan tipe data yang ingin dimanipulasi saat runtime. Terkadang saya temukan pengecekan tipe data ini dilakukan di tempat yang kurang tepat.

```js
const allowedProps = ['title', ...Object.keys(Tooltip.propTypes)]
```

Di sini si programmer terlihat "malas". Ia malah mencoba untuk meng-copy semua key yang ada di `Tooltip.propTypes`. Yang lucu dari kasus ini adalah ternyata object `Tooltip.propTypes` hanya dibuat ketika development time, alias saat production build nilainya akan menjadi undefined 😄

```sh
> Uncaught TypeError: Cannot convert undefined or null to object
```


Walhasil aplikasi error saat runtime dan agak tricky ketika melakukan debugging. Saya terpaksa harus membuka source code Tooltip Material UI dulu baru kemudian sadar object ini bernilai undefined on production. Niatnya type-safe malah berujung bencana 🤦🏼‍♂️

Pernah juga saya mendapatkan code seperti ini:

```js
const someFunc = (key, value) => {
  const msg = {
    undefinedKey: 'key must not be undefined',
    keyShouldBeArray: 'key should be an array',
    valueShouldBeArray: 'value should be an array',
  };

  if (key === undefined) throw new TypeError(msg.undefinedKey);
  if (!Array.isArray(key)) throw new TypeError(msg.keyShouldBeArray);
  if (!Array.isArray(value)) throw new TypeError(msg.valueShouldBeArray));

  // run something..
}
```

Belum lagi kalau ada unit test..

```js
it('accepts no string', () => { ... })
it('should throw an error when `key` is not array', () => { ... })
it('should throw an error when `value` is not array', () => { ... })
```

Yang sebenarnya sah-sah saja, bagus malah, **kalau sedang membuat library**, karena gak semua user menggunakan static analysis tool seperti Typescript atau Flow sementara mereka harus tau apa jenis kesalahannya. Kalau sekedar aplikasi biasa, saya rasa pengecekan-pengecekan semacam ini bisa didelegasikan ke compiler lewat types. Kita gak perlu invent our own type-checker 🙃

```ts
const someFunc = (key: string[], value: string[]) => {
  // run something..
}
```

## Kenapa Functional Programming?
Di sini saya gak akan menjelaskan panjang-lebar kenapa kita sebagai programmer harus setidaknya mencoba belajar konsep Functional Programming &mdash; jawaban-jawaban tersebut pasti banyak temen-temen temukan di Google.

Alasan pribadi saya: Functional Programming makes more sense to me. _Composition_, _pure functions,_ dan _referential transparency_ adalah tiga kata kunci kenapa saya lebih memilih Function Programming ketimbang OOP. OOP pada umumnya menggunakan _class_ sebagai building block untuk membangun program, dimana dengan menggunakan _class_ sendiri saja menurut saya sudah membuka jalan untuk [melakukan mutasi state]({{< ref "/post/kenapa-immutability-itu-penting-javascript.md" >}}) yang mengakibatkan sulit tercapainya _referential transparency_. Debugging menjadi lebih sulit dan memaksa saya untuk benar-benar mengikuti alur program agar tidak terjadi hal-hal yang tidak diinginkan karena perubahan state. Alasan ini murni dari pengalaman pribadi dan bisa jadi subjektif ya 🙂

BTW, beberapa bahasa FP yang tidak memiliki static-typing diantaranya ada Racket, Clojure, Scheme, atau Common Lisp (dari LISP family). Elixir juga sepertinya sudah mulai banyak yang adopsi. Bahasa-bahasa ini mungkin lebih cocok untuk dipelajari bagi yang ingin mendalami Functional Programming namun tidak terlalu ingin dipusingkan dengan types.

## Kenapa Purescript?
Balik ke kalimat awal: karena mau explore _typed functional programming_ lebih dalam lagi. Bagi kamu yang belum tahu apa itu Purescript, [Purescript is a strongly-typed functional programming language that compiles to JavaScript](http://www.purescript.org/). Bahasa ini sangat terinspirasi dari Haskell dari segi syntax, paradigma, dan fitur-fitur static typing-nya. Compiler-nya bahkan ditulis menggunakan Haskell!

Ada beberapa alasan pribadi kenapa saya lebih memilih Purescript untuk mendalami _typed functional programming_ dibandingkan dengan bahasa-bahasa lainnya:

### Setup Gak Ribet
Sebelum lari ke Purescript, saya sempet belajar beberapa konsep typed FP dengan Haskell.
Kesan saya setelah beberapa kali setup project di Haskell sebagai seorang nubi itu: ribet! Banyak hal yang perlu di-setup, mulai dari pilih-pilih editor (Vim, Emacs, VS code?) sampai backend engine (Intero, HIE, GHCid?). Sempet pakai HIE juga namun semangat mulai luntur begitu language extention `TemplateHaskell` diaktifkan, karena HIE langsung kejang-kejang. Build time untuk setup HIE pun bisa sampai setengah jam, memaksa laptop saya merangkap jabatan sebagai helikopter. Belum lagi disk space yang bakal banyak dimakan sama GHC 🤯

Setup projek di Purescript sendiri justru sangat friendly. Cukup jalankan

{{< highlight bash >}}
$ npm i -g purescript spago

$ mkdir your-project
$ cd your-project

$ spago init
$ spago build
{{< /highlight >}}

dan projek sudah langsung ready 🙂 Untuk editor saya tetap menggunakan VS Code dengan [Purescript IDE](https://github.com/nwolverson/vscode-ide-purescript) sebagai pluginnya.

Jadi bagi pemula yang ingin langsung belajar _typed functional programming_ tanpa perlu keluar effort setup sana-sini saya sarankan untuk belajar Purescript!

### Community Support
Dengan komunitas yang masih belum terbilang besar, belajar Purescript tidak serta merta menjadi hal yang sulit dilakukan. Saya justru sering mengikuti diskusi-diskusi berkualitas di channel <mark>#purescript</mark> ([link di sini](https://fpchat-invite.herokuapp.com/)) dari para core contributor Purescript seperti Mas Nate Faubion, Pak Harry, Om Thomas Honeyman, Pakde Harry Garrood, dan Aa' Christoph Hegemann. Untuk pertanyaan-pertanyaan nubi, bisa ditanyakan ke channel <mark>#purescript-beginners</mark> dijamin fast-response 😉

### Hindley-Milner Type System
Salah satu keunggulan dari bahasa pemrograman keluarga ML adalah penggunaan type system Hindley-Milner, dimana programmer tidak perlu memberikan type annotations di setiap variable atau function. In most cases the compiler will just infer it for you. Sebagai perbandingan sederhana, compiler Typescript belum bisa meng-infer type dari variable yang muncul di function arguments:

```js
const mult = (x, y) => x * y;
              ↑  ↑
             any any
```

Di Purescript, compiler _automagically_ tahu bahwa `x` dan `y` **sudah pasti** memiliki tipe Number dan akan memberikan pesan error jika kita masukkan, misal, sebuah string ke dalam function tersebut. Walaupun sebenarnya bisa-bisa saja membuat function tanpa type signature, namun penulisannya masih tetap disarankan sebagai bentuk "living documentation" bagi diri sendiri dan developer lain.

```hs
mult :: Number -> Number -> Number
mult x y = x * y
```

Dan power dalam meng-infer types ini bisa juga kita "eksploitasi" ketika otak sudah nggak bisa diandalkan lagi. Purescript mempunyai fitur _type hole_ yang memungkinkan programmer untuk "bertanya" ke compiler apa type atau function yang cocok digunakan di bagian aplikasi tertentu. Sebagai contoh, saya punya function `program` dan saya ingin compiler meng-infer type function tersebut.

```hs
program :: ?help
program = command "cat" (info runCat $ progDesc "Simply read a file")
```

Compiler akan melakukan analisis struktur program, mendeduksi types-nya dan bimsalabim jadi apa prok-prok-prok:
{{% figure src="/uploads/type_hole_answer-type.png" alt="type-hole answer type" caption="Inferred type" class="fig-center img-60" %}}

Tinggal di-copy-paste saja jawaban dari compiler :)) Ohiya, type hole ini tidak hanya berguna untuk mengetahui suatu type saja, tapi juga bisa dimanfaatkan untuk mencari tau function apa saja yang compatible dengan program kita. Mari gunakan code snippet yang sama dan ganti `command` dengan `?help`.

```hs
program :: Mod CommandFields (AppM Unit)
program = ?help "cat" (info runCat $ progDesc "Simply read a file")
```

Compiler akan melakukan tugasnya untuk mencari function yang cocok menggantikan placeholder `?help`. Dan bisa dilihat function `command` yang kita inginkan ternyata ada di suggestion nomor 2 👀

{{% figure src="/uploads/type_hole_answer.png" alt="type-hole answer" caption="Compatible functions" class="fig-center" %}}

> 💡Fun fact: fitur "values suggestion" ini dinamai Type-Directed Search, yang merupakan hasil dari tugas akhir (skripsi) Mas Christoph Hegemann berjudul **Implementing Type-Directed Search for Purescript** yang disimpulkan lewat [PR ini](https://github.com/purescript/purescript/pull/2352/files).

Terima kasih compiler 🤗🤗🤗

### Higher-Kinded Type dan Typeclass
Konsep ini pernah saya singgung [di post tersendiri]({{< ref "/post/generic-di-atas-generic-higher-kinded-type.md" >}}) beberapa waktu yang lalu.

TL;DR: Higher-Kinded Type menyediakan kemampuan mengabstraksi type constructor sehingga code yang ditulis tidak lagi terikat dengan implementation details. Yang saya suka dari konsep ini adalah code menjadi super reusable dan generic. HKT ini jugalah yang membuat konsep Typeclass jadi lebih make sense, semua untuk mencapai code reusability.

Ada video bagus dari Nate Faubion tentang bagaimana cara me-refactor function biasa menjadi function yang generic dengan bantuan Higher-Kinded Type. Video ini **sangat saya rekomendasikan** bagi siapa saja yang tertarik dengan code reusability 😃

{{< youtube GlUcCPmH8wI >}}

<br />

### Algebraic Data Type (ADT) dan Pattern Matching
Algabraic Data Type adalah sebuah tipe data yang dibangun dari dua buah konstruksi: **Sum** dan **Product**.

Untuk _Sum_ sendiri, ia digunakan untuk merepresentasikan suatu data yang dapat memiliki beberapa varian. Anggap saja ini sebagai "or" operator.

```hs
-- Sum type
data Boolean = True | False -- true `or` false

data Color = Red | Green | Blue -- red, `or` green, `or` blue
```

Sedangkan _Product_ &mdash; kebalikan dari _Sum_ &mdash; digunakan untuk menggabungkan dua atau lebih tipe data ke dalam satu varian. Anggap saja ini sebagai "and" operator.

```hs
data TupleIntAndString = Tuple Int String -- Int & String

-- alias
type Base = Number
type Width = Number
type Height = Number
type Side = Number

data Shape
  = Triangle Base Height -- Base & Height
  | Rectangle Width Height -- Width & Height
  | Square Side
```

Dan biasanya bahasa pemrograman yang sudah memiliki fitur ADT juga memiliki fitur Pattern Matching untuk mendekonstruksi ADT tersebut.

```hs
-- cara 1
printColor :: Color -> Effect String
printColor = case _ of
  Red   -> log "red"
  Green -> log "green"
  Blue  -> log "blue"

-- cara 2
printColor :: Color -> Effect String
printColor Red   = log "red"
printColor Green = log "green"
printColor Blue  = log "blue"
```

Saya sendiri sih suka banget sama fitur ADT terlebih ketika menyinggung ranah domain modelling, code menjadi lebih ekspresif dan mudah dibaca. Di bahasa OOP, ADT dapat diekspresikan dengan simple inheritance relationship between Parent and Child class, alias subtype polymorphism. Saya juga pernah menulis artikel tentang [perbandingan antara subtyping dengan ADT beserta kelebihan dan kekurangannya](https://medium.com/codewey/subtype-polymorphism-vs-algebraic-data-types-adt-94a0f186b8d1).

### Foreign Function Interface (FFI)
FFI memungkinkan Purescript berkomunikasi dengan bahasa target (dalam hal ini Javascript, walaupun bisa juga di-porting ke [C++](https://github.com/andyarvanitis/purescript-native) atau [Erlang](https://github.com/purerl/purescript)). Dengan FFI kita bisa memanggil code di Javascript dari Purescript!

```js
// javascript
exports.add = function (x) {
  return function (y) {
    return x + y;
  }
}

exports.argv = process.argv;
```

```hs
-- purescript
foreign import add :: Number -> Number -> Number
foreign import argv :: Array String

increment = add 1
decrement = add -1
```

Begitupun sebaliknya, pemanggilan code Purescript dari Javascript juga mungkin dilakukan. Pembahasan lengkapnya ada di buku [Purescript by Example](https://leanpub.com/purescript/read#leanpub-auto-chapter-goals-8) (free).

### Explicit Side Effect
Purescript sebagai _pure functional language_ membatasi programmer dalam hal pemanggilan function yang effectful. Artinya kita tidak bisa sembarangan melakukan side-effect seperti [mutasi variable]({{< ref "/post/kenapa-immutability-itu-penting-javascript.md" >}}), pemanggilan HTTP request, akses DOM, generate random numbers, akses Local Storage, sampai hal sesimple logging ke terminal. Hal-hal yang berpotensi mengubah "state" di luar lingkup suatu function harus dibuat eksplisit lewat type signature. Walhasil, kita sebagai programmer tahu dengan sangat jelas mana function yang pure dan mana function yang effectful, umumnya ditandai dengan `Effect` monad.

```hs
readFile :: FilePath -> Effect String
readFile path = ...
```

Sehingga hampir "mustahil" bagi kita untuk membuat function yang effectful tanpa diketahui oleh compiler. Strictness inilah yang memaksa programmer untuk membuat program yang predictable dan sebisa mungkin effect-free agar behaviour yang tidak diinginkan dapat terminimalisir. Konsekuensinya kita harus banyak "berantem" dulu sama compiler alih-alih menghabiskan waktu untuk debugging nantinya. Hmm, sepertinya Purescript lebih memilih Correctness over Convenience..

## Caveats
Kelebihan-kelebihan tersebut juga harus dibayar dengan beberapa kekurangan. Misalnya pesan error yang terkadang tidak begitu jelas. Akan sangat terasa sekali kalau sudah mulai banyak menggunakan Monad dan function yang generic.

```
Could not match type

    t2 Unit

  with type

    Unit


while trying to match type t0 t1
  with type AppM Unit

where t1 is an unknown type
      t0 is an unknown type
      t2 is an unknown type
```

Dari sudut pandang saya pribadi yang "cuman mau belajar" justru lumayan menantang dan encouraging 😄, benar-benar memutar otak untuk mencari tahu why-nya dan nggak sekedar main tebak-tebakan sama compiler.

Kemudian learning resource-nya juga masih terbilang sedikit. Kebanyakan _technique_ yang saya gunakan di Purescript justru berasal dari hasil belajar saya dengan Haskell karena kemiripan kedua bahasa ini. Hal ini bisa jadi kelebihan juga sebenarnya, if you can't find something in Purescript then look for one in Haskell, you'll probably end up getting the answer 😅

## Epilog
Sejauh ini kesan saya belajar Purescript masih positif. Teman-teman bisa pantau [projek sampingan saya di Github](https://github.com/dewey92/pureshell) sebagai progres saya belajar Purescript. Sekian jam dari awal pengerjaan projek sampai setidaknya penulisan artikel ini (sekitar 42-an jam, tracking dari wakatime) saya belum pernah menemukan runtime error seperti "Undefined is not a function", "Cannot convert undefined or null to object", atau error-error lainnya yang umum didapati ketika mengembangkan aplikasi menggunakan Javascript.

Purescript is a safe language, dengan segala bentuk konsekuensinya. Static typing-nya mungkin akan sering membuat kita frustasi, tapi akan selalu berujung pada apresiasi; terutama saat runtime. Yang senang senam otak saat development time mungkin suatu saat bisa mencoba Purescript haha. Selamat malam ✌🏻😃