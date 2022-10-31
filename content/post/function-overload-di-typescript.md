---
title: "Function Overload di Typescript"
date: 2022-10-28T19:57:09+02:00
description: "Function overload memungkinkan kita untuk mendefinisikan kombinasi type yang bervariasi baik di posisi parameter maupun di posisi return."
images: ["/uploads/overload.jpg"]
image:
  src: "/uploads/overload.jpg"
  caption: Image by <a href="https://pixabay.com/users/shaq64-14977059/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4780853">Stephan Westphal</a> from <a href="https://pixabay.com//?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4780853">Pixabay</a>
tags: ["typescript", "types"]
categories: ["programming"]
draft: false
---

Misal kita ingin membuat function `combine` untuk menggabungkan dua buah string atau array

```ts
function combine(a, b) {
  return a.concat(b)
}

combine('jihad', 'waspada')     // 'jihadwaspada'
combine(['jihad'], ['waspada']) // ['jihad', 'waspada']
combine('jihad', ['waspada'])   // 'jihadwaspada'
combine(['jihad'], 'waspada')   // ['jihad', 'waspada']
```

Keliahatannya oke, tapi tidak dengan

```ts
combine('jihad', ['waspada', 'ganteng'])   // 'jihadwaspada,ganteng'
```

Lho kenapa ada tanda komanya? Harusnya kan `'jihadwaspadaganteng'`? Entahlah. Mungkin ini salah satu keajaiban javascript. Tapi yang pasti kita nggak pingin dong kodenya menghasilkan sesuatu yang tidak reliable (tidak intuitif, tidak sesuai ekspektasi).

Untuk memperbaikinya mungkin bisa kita batasi requirement fungsi `combine` dimana:
1. Jika argument pertamanya berupa string, argument keduanya juga harus string. Return type-nya juga string
2. Jika argument pertamanya berbentuk array, argument keduanya juga harus array. Return type-nya juga array

Di luar kombinasi ini—seperti `combine('jihad', ['waspada'])` atau `combine(['jihad'], 'waspada')`—harus error, nggak typecheck. Lalu gimana cara mengekspresikan requirement ini dengan Typescript?

Kita coba dulu pake union biasa:

```ts
function combine<T>(a: string | T[], b: string | T[]): string | T[] {
  return a.concat(b)
                  ^
  // Type 'string' is not assignable to type '(T | combineArray<T>) & string'
}
```

Hmm error. Di sini typechecker berusaha melakukan unifikasi `string` dengan `T[]` di posisi contravariant (parameter method `concat`) menghasilkan `(T | combineArray<T>) & string` tapi gak typecheck ketika dikasih argument berupa string. Penjelasan lebih detailnya dibahas di [artikel ini]({{<ref "./covariance-and-contravariance-in-typescript.md" >}}). Tapi bukan itu masalah utamanya.

Masalahnya utamanya, type signature fungsi `combine` masih ill-typed sekalipun implementasinya typecheck:

```ts
combine('jihad', ['waspada']) // typecheck 🤨
combine(['jihad'], 'waspada') // typecheck 🤨
```

Kode di atas seharusnya error tapi malah typecheck. Lha terus piye?

Kita ingin membuat asosiasi type di function parameter dengan return type-nya. Dan di sinilah fitur _function overload_ Typescript berguna! Type signature-nya pun cukup ekspresif:

```ts
function combine   (a: string, b: string): string
function combine<T>(a: T[]   , b: T[]   ): T[]
function combine   (a: any   , b: any   ) {
  return a.concat(b)
}
```

Kita hanya perlu menuliskan type annotations di posisi parameter dan return untuk melakukan function overload, tak perlu function body. Typechecker akan meng-evaluasi overload yang paling atas dulu dan akan lanjut ke overload di baris selanjutnya bila yang pertama tadi gak cocok. Begitupun seterusnya sampai mentok di overload terakhir. Untuk type annotation di function implementation-nya sendiri, cukup kita definisikan dengan type yang kompatibel dengan semua overload-nya: dalam kasus ini saya pilih `any` biar gampang.

Oleh karena itu ketika menuliskan function overload, type yang paling spesifik sebaiknya diletakkkan di baris teratas.

Dan ketika kita check:

```ts
const res1 = combine('jihad', 'waspada')
// ✅ res1: string
const res2 = combine(['jihad'], ['waspada'])
// ✅ res2: string[]

combine('jihad', ['waspada']) // ❌ No overload matches this call
combine(['jihad'], 'waspada') // ❌ No overload matches this call
combine(1, 2) // ❌ No overload matches this call
```

Naiiss 🔥

{{% message success %}}
  Function overload memungkinkan kita untuk mendefinisikan kombinasi type yang bervariasi baik di posisi parameter maupun di posisi return.
{{% /message %}}
---

Saya punya contoh lain: anggap kita ingin membuat fungsi `findElem`, mirip-mirip dengan `Array.find` tapi return type-nya bisa dikonfigurasi jika item yang dicari tidak ditemukan. Ada 3 konfigurasi:
1. Mengembalikan undefined
2. Mengembalikan error message
3. Mengembalikan default value

```ts
type WithError<E> = { withError: E }
type WithDefault<T> = { withDefault: T }

type Config<E, T> =
  | undefined
  | WithError<E>
  | WithDefault<T>
```

Dengan konfigurasi ini, kita bisa membuat fungsi `findElem` seperti berikut:

```ts
function findElem<T>   (items: T[], item: T): T | undefined
function findElem<T>   (items: T[], item: T, config: WithDefault<T>): T
function findElem<E, T>(items: T[], item: T, config: WithError<E>  ): T | E
function findElem<E, T>(items: T[], item: T, config?: Config<E, T> ) {
  const result = items.find((i) => i === item)

  if (config === undefined) return result
  if ('withError' in config) return result ?? config.withError
  return result ?? config.withDefault
}

const res1 = findElem([1, 2], 3)
// ✅ res1: number | undefined
const res2 = findElem([1, 2], 3, { withError: new Error('Cannot find elem') })
// ✅ res2: number | Error
const res3 = findElem([1, 2], 3, { withDefault: 9 })
// ✅ res3: number
```

Sama seperti fungsi `combine` di pembahasan sebelumnya dimana return type-nya bergantung kepada type di kedua buah parameter-nya, return type fungsi `findElem` bergantung kepada type `Config` yang disuplai.

Menyuplainya dengan config yang tak ada di-definisi overload menyebabkan error:

```ts
const notTypecheck = findElem([1, 2], 3, {
  withError: new Error('Cannot find elem'),
  withDefault: 9,
})
// ❌ No overload matches this call
```

---

Saya harap artikel ini bermanfaat.
