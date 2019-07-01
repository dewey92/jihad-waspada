---
title: "Kenapa Immutability Itu Penting (Javascript)"
date: 2019-06-30T19:56:30+02:00
description: Dalam banyak kasus, Immutability justru membantu menghilangkan kompleksitas yang sebenarnya tidak perlu
images: ["/uploads/immutable.jpg"]
image:
  src: "/uploads/immutable.jpg"
  caption: Image by <a href="https://pixabay.com/users/Monsterkoi-65294/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2817950">Monsterkoi</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2817950">Pixabay</a>
tags: ["javascript", "programming"]
categories: ["javascript"]
draft: false
---

Beberapa hari yang lalu PO kami menemukan bug yang cukup unik di salah satu projek legacy kami dimana setelah user meng-upload foto profile-nya, foto tersebut akan tampil dan langsung menghilang sepersekian detik kemudian. Kayak baru kenal tapi langsung diputusin gitu 😕 Saya pun melakukan Pair Programming dengan temen satu team selama kurang lebih setengah jam. Dataflow-nya oke, redux action gak ada yang masalah, payload dari/ke server pun fine-fine saja. Hmm. Sampai momen dimana kami menemukan satu baris kode yang **kelihatannya oke tapi gak oke**.

```js
const newState = { ...state }
newState.isRegistered = true
delete newState.profile.picture // <- This guy!
```

Langsung aja kami misuh-misuh di tempat, "_This is such a ridiculous bug!_ 💩💩💩😡🤬🤬🤬"

# Const dan Let
Sebelum memahami kenapa kami bisa menyimpulkan code di atas adalah biang masalahnya, saya mau mengulas dulu apa sih Immutability itu. <mark>Immutability dalam programming adalah suatu value yang tidak bisa diubah ketika sudah dideklarasikan</mark>. Perhatikan potongan code berikut

```js
const name = "Jihad"
name = "Dzikri"
```

Javascript akan complain bahwa variable `name` tidak dadat diubah: `Uncaught TypeError: Assignment to constant variable`. Mirip-mirip begitu lah. Selama saya bekerja dengan Javascript tiga tahun belakangan, saya hampir-hampir tidak pernah menggunakan `let` dan lebih memilih `const`. Sekedar menghindari _mutability_.

```js
let i = 9

console.log(i + 1 === i + 1)
console.log(i++ === i++)
```

Kira-kira apa jawaban log yang pertama dan apa jawaban log yang kedua?

Log yang pertama akan bernilai `true` karena keduanya bernilai 10 `console.log(10 === 10)`. Operasi ini bersifat _immutable_ karena tidak ada variable yang diubah ketika runtime.

Tapi log yang kedua akan bernilai `false` karena operasi yang satu ini bersifat _mutable_: nilai `i` berubah-ubah. Ketika Javascript menjalankan `i++` pertama, nilai `i` berubah menjadi 10.

```js
console.log(10 === i++)
```

Setelahnya, nilai `i` akan berubah lagi menjadi 11 disebabkan oleh statement `i++` yang kedua.

```js
console.log(10 === 11) // FALSE!
```

Sampe sejauh ini kita paham bahwa `const` bisa digunakan ketika kita ingin variable tersebut tidak bisa diganti, dan `let` bisa digunakan ketika ada variable yang ingin diganti _over time_.

# Object di Javascript
## Flat Object
Gak selamanya variable yang dideklarasikan menggunakan `const` itu nggak bisa berubah. Iya, Javascript ini emang rada-rada gaes. Contohnya gimana, Mas Jihad?

```js
const user = {
  firstName: 'Jihad'
}

// error! Uncaught TypeError: Assignment to constant variable
user = {
  firstName: 'Dzikri'
}

// gak error
user.firstName = 'Dzikri'
```

Bisa jadi fatal sekali kalau kita nggak _aware_ sama _behaviour_ ini. Gak jarang saya temui beberapa junior developer atau bahkan sudah bisa dibilang mid-level tapi tetap melakukan **mutasi** seperti di atas tanpa sadar akan konsekuensinya.

```js
function deleteFirstName(user) {
  user.firstName = undefined
  return user
}

const jihad = {
  firstName: 'Jihad',
  lastName: 'Waspada',
  age: 26,
}
const userWithoutName = deleteFirstName(jihad)

console.log('No name:', userWithoutName.firstName)
console.log('Jihad: ', jihad.firstName)
console.log(userWithoutName === jihad)
```

```
> No name: undefined
> Jihad: undefined
> true
```

Kok bisa?? Bukannya yang satu harusnya `undefined` dan yang satunya tetep `'Jihad'`?? Kok dua-duanya `undefined`??

> "Mereka kira mereka bisa menjawab sedangkan mereka termasuk orang-orang yang tidak tahu" - JS 1:12

Keduanya bernilai `undefined` karena <mark>secara default, Object dalam Javascript sifatnya _pass by reference_, bukan _pass by value_ ketika dilempar ke dalam suatu function/method</mark>. Jadi sebenarnya variable `jihad` dan `userWithoutName` adalah variable yang sama (_point to the same address_), hanya namanya saja yang berbeda. Untuk mengakalinya, kita harus ubah sedikit dengan object destructuring atau spread operator.

```js
function solusi1(user) {
  const noFirstName = { ...user }
  noFirstName.firstName = undefined
  return noFirstName
}

// atau

function solusi2(user) {
  return {
    ...user,
    firstName: undefined,
  }
}

// ...

console.log('No name:', userWithoutName.firstName) // undefined
console.log('Jihad: ', jihad.firstName) // 'Jihad'
console.log(userWithoutName === jihad) // false
```

_Now it works.._

> Saya pribadi lebih suka `solusi2` dan menghindari sebisa mungkin `solusi1`. Dan saya rekomendasikan teman-teman untuk pakai `solusi2` sebisa mungkin supaya gak dituduh orang yang tidak bertangungjawab. Hehe canda, biar lebih mudah aja kalau punya nested object 👇🏻

## Nested Object

Prinsip di atas bisa diaplikasikan juga untuk **Object di dalam Object**. Karena sejatinya operasi `{ ...user }` tidaklah cukup jika object `user` memiliki object lagi.

{{< highlight js "linenos=inline,noclasses=false,hl_lines=2 10-13" >}}
function removeProfilePicture(user) {
  const noPicture = { ...user }
  noPicture.profile.picture = undefined

  return noPicture
}

const jihad = {
  firstName: 'Jihad',
  profile: {
    picture: '/uploads/orang_ganteng.jpg',
    userName: 'dewey992'
  }
}

const userWithoutPicture = removeProfilePicture(jihad)

console.log('no picture: ', userWithoutPicture.profile.picture)
console.log('jihad: ', jihad.profile.picture)
{{< /highlight >}}

```
> no picture: undefined
> jihad: undefined
```

Keduanya lagi-lagi bernilai `undefined` karena pada code di baris ke-2 **hanya membuat Object baru di level pertama saja**. Level berikutnya (`profile.picture` dan `profile.userName`) akan tetap menunjuk pada reference sebelumnya. Solusinya adalah dengan membuat object baru lagi!

```js
function solusi3(user) {
  return {
    ...user,
    profile: {
      ...user.profile,
      picture: undefined,
    }
  }
}
```

Mirip dengan `solusi2`. Dan alasan inilah kenapa saya lebih suka "style" `solusi2` dibandingkan `solusi1` karena code-nya lebih straighforward dan terhindar dari _any possible bugs_ yang diakibatkan oleh mutasi object.

## Array
Aturan di atas juga berlaku untuk Array karena pada dasarnya Array adalah object 🤔

```js
console.log(typeof [])
// > "object"
```

Dibilang Javascript ini rada-rada. Tapi intinya, diperlukan kehati-hatian juga dalam hal ini.

{{< highlight js "noclasses=false,hl_lines=4" >}}
const persons = [{ age: 23 }, { age: 25 }]

const newPersons = persons.map(person => {
  person.age = 1 // Halo gaes!
  return person
})

console.log(persons)    // [{ age: 1 }, { age: 1 }]
console.log(newPersons) // [{ age: 1 }, { age: 1 }]
console.log(persons === newPersons) // FALSE!
{{< /highlight >}}

# Intermezzo dengan React
Satu kasus yang cukup simple dimana _mutability_ bisa mengakibatkan kita garuk-garuk kepala, mikir keras kenapa component kita gak jalan sesuai yang diharapkan. Mari berasumsi ada sebuah component yang gemuk dan _expensive_ dari segi rerendering sehingga kita perlu mengimplementasikan method `shouldComponentUpdate`

```js
class ExpensiveComp extends React.Component {
  // ...

  shouldComponentUpdate(nextProps) {
    if (nextProps.user !== this.props.user) return true;
    return false;
  }

  // ...
}
```

Jika object `user` diubah dengan cara yang _mutable_, component tersebut gak akan pernah bisa melakukan rerendering karena object `user` yang baru dianggap sama dengan yang lama ⇒ bisa-bisa gak reaktif sama sekali. Immutability dalam hal ini membantu menghilangkan kompleksitas-kompleksitas yang sebenarnya tidak perlu.

# Penutup
Sekarang kita sudah cukup paham _behaviour_ Object di Javascript yang memiliki nature _pass by reference_ 🎉 Ada beberapa keuntungan yang didapat jika menghindari mutasi variable dan object.

- Sadar atau tidak sadar, ketiga function di atas (`solusi1`, `solusi2`, `solusi3`) semuanya adalah _pure function_. Yang dimaksud dengan _pure function_ adalah function yang tidak mengubah nilai di luar scope-nya. Ketiga function tersebut tidak mengubah object `user`, mereka justru **mengembalikan object baru**.
- Karena _pure function_ inilah, Kita bisa mencapai [Referential transparency](https://en.wikipedia.org/wiki/Referential_transparency). Sehingga nggak akan ada ceritanya suatu function ngebuat error di bagian aplikasi yang lain yang sama sekali gak ada hubungannya sama function ini. _Unknown side effects are always evil_.
- Dan yang paling penting: memudahkan proses debugging! Gak pingin kan dijadiin bahan cacian sama developer lain yang maintain code kita nantinya hanya karena **rookie mistake** begini.. 🤪
- Di beberapa bahasa yang support multithreading semacam Java atau Scala, Immutability dapat [menghindari program dari _race condition_ dan berjalan di thread yang safe](https://pasztor.at/blog/why-immutability-matters)

Saran saya pribadi: kalau _mutability_ bisa dihindari, hindari saja. Kalau memang tidak bisa dihindari karena alasan-alasan tertentu, pastikan scope-nya tidak terlalu besar agar kedepannya lebih mudah di-debug.

Semoga bermanfaat 🙂