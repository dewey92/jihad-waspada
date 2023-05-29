---
title: "Existential Type di Typescript"
date: 2023-05-22T19:05:40+02:00
description: "Bagaimana kalau ternyata kamu bisa membuat private sebuah type? Hanya bisa diakses oleh implementor namun tidak oleh consumer?"
images: ["/uploads/hide.jpg"]
image:
  src: "/uploads/hide.jpg"
  caption: Image by <a href="https://pixabay.com/users/ambermb-3121132/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1635065">ambermb</a> from <a href="https://pixabay.com//?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1635065">Pixabay</a>
tags: ["typescript", "types"]
categories: ["programming", "type system"]
draft: false
---

## Motivasi

{{% message info %}}
  Untuk kamu yang suka langsung liat kodenya, inspect apa type-nya, gimana strukturnya, dll selagi baca teknikal blog, saya lampirkan link kodenya di [TS Playground ini](https://www.typescriptlang.org/play?ssl=21&ssc=45&pln=21&pc=66#code/JYWwDg9gTgLgBAKjgQwM5wEoFNkGN4BmUEIcA5FDvmQFA0AmWuANspXLhAHarwASORlABcmKjAB0AYRKQuWLjAAqATzBYGTVu0494ABWIFgzLKOx5JM8NwXK1WADwBvOAFdUWKF2QgzcXihgLgBzOABfAD5NFjYsDm5eOABlYEYAIzZzcWlZW0VVdRc4dOYIEMheVFFA4JCAbQBdCOiaGAc4KQALNy4Aa0d9SLgAXjh6kKwYQwgwargACgBKUeGZkGBPQciAGgTwbMtcm3kCh23GukZY9h8-VDA8eIBBMGA4Zxo4OCwAD0hYAk9HBJtMjCYsDM5qJlqs4OtNk5XB4vHd-LVQi0vj9-tB4LokqDUhk2FD5rCRmtiBstq5SuVKjB5hiGs0ojRwnQAPRc0YjfkCwVC4Ui0Vi8USkY0HlwACqqDqnX0yWlvMl6o1mo1dHa6k6PX6UmVozgjgwkQW8l+MFEgxxMAU9HQziiC1wBr6om6vQGQxWlMw-uGGDaHSklGQDu9-RNdr+Dq4To+roA+l6Pdsg-qfUaVTQCfiI1GPV6i1ho30TW6PVnLfGs1aYNWfUsdWGPagALLIMAm7tgRwsvYV3OtAscDvpn1dnsm+QAdzg-eW+Y7Ek8TbIXUEXjIe1wZYrC3qFKpJEREkoqAgzAAblgFi6lnsBMghI0lq33dP11MFmQwHBUw9w4Q8PWPU84FeYAJFBGZjFMMllj2eCIQ-L81w3f8FRJKAQIPHBix9CCs2g2CpmJLBMigJDnxSNIqLYdC6G-fpUAkAhoAAUTwLoFgWXp5ygHs9j6LAVCzT5vnHLBmBNegIFwNw-EUciYC40wVJgAAhFQAEl6AWMSJOxYACEWABCWSVkoGA3G8OhvkE4SwAWAhenwYBuDgRtm36FYpO+IEknqAgpndMl91kZoxlYvpsW+MKYAi4g5mWCQYG3Lh+MA2ZUCzCx8AkAjIywDSsC0t1ZD2XK5k-bFwlbRqgA).
{{% /message %}}

Pernah gak sih kamu pengen make generic type tanpa harus menyuplai parameternya dengan type lain? Mungkin rada abstrak kali ya, tapi coba deh bayangin kamu punya tuple dimana komputasi di item pertama bakal dipake sebagai argument di item yang kedua:

```ts
type Chunk<P> = [getProps: () => Promise<P>, comp: React.ComponentType<P>]
```

Lalu chunk-chunk ini bakal disimpan di dalam hashmap:

```ts {hl_lines=[1]}
type ChunksMap = Map<string, Chunk<any>>
                                   ^^^

const chunks: ChunksMap = new Map()
chunks.set('header', [() => Promise.resolve({}), Header])
chunks.set('profile', [() => Api.getProfileProps(), Profile])
chunks.set('sidebar', [() => Api.getSidebarProps(), Sidebar])
```

Nah sekarang kebayang kan maksudnya, kamu cuman mau make type `Chunk` di `ChunksMap` tanpa harus mengisi parameter `Chunk`. Kita gak bisa buat type parameter `P` untuk `ChunksMap` (`type ChunksMap<P> = ...`) dan harus fallback ke `any` karena masing-masing `Chunk` dapat memiliki instance `P` yang berbeda-beda; `P` bisa berupa `{}`, `ProfileProps`, atau `SidebarProps`.

Padahal tau sendiri kan `any` gak boleh diandelin di sini karena, misal, `Api.getSidebarProps()` jadi punya potensi untuk ngisi props-nya component `Profile`. Big no.

Andai saja Typescript menyediakan suatu mekanisme yang memungkinkan untuk bilang, "yo type checker, ini ada suatu type yang dibutuhkan `Chunk`, tapi gw gak tau detail type-nya. Yang gw tau dia _ada_ dan dipake", mungkin kodenya akan tampak lebih ekspresif.

```ts
type ChunksMap = Map<string, Chunk<exists P>>
```

Inilah yang dimaksud dengan _existential type_. Walau Typescript belum mendukung fitur ini, bukan berarti gak ada cara lain untuk ngakalinnya! _Existential type_ bisa di-encode dengan CPS (continuation-passing style).

## Solusi

Singkatnya function yang ditulis menggunakan gaya CPS menerima satu argument tambahan berupa function lain (_continuation function_) yang akan memproses hasil komputasi function yang pertama tadi. Ekspresi `(1 + 2) * 3` bila ditulis menggunakan CPS akan menjadi

```js
const add = (x, y) => (next) => next(x + y)
const mul = (x, y) => (next) => next(x * y)

const run = (next) => add(1,2)((r) => mul(r, 3)(next))
run(console.log)
```

Bila style ini diterapkan untuk meng-encode _existential type_:

```ts
const createChunk: CreateChunk = (chunk) => (next) => next(chunk)

type CreateChunk = <P>(_: Chunk<P>) => ChunkCPS
type ChunkCPS = <R>(next: <P>(_: Chunk<P>) => R) => R
```

Ada hal yang menarik dengan type `CreateChunk` dan `ChunkCPS`:
1. Keduanya tak memiliki type parameter, karena...
2. Deklarasi type `P` dan `R` berada di <abbr title="right hand side, sebelah kanan persamaan">RHS</abbr>.

<details>
  <summary>RHS? LHS?</summary>
  Umumnya generic type ditulis di LHS (sebelah kiri persamaan) sebagai type parameter

  ```ts
  type Identity<T> = (val: T) => T
  ```

  Namun ia bisa juga ditulis di sebalah kanan persamaan

  ```ts
  type Identity = <T>(val: T) => T
  ```
</details>

Di Typescript, kemampuan mendeklarasikan generic type di RHS hanya bisa dilakukan oleh function. Di bahasa lain seperti Haskell, cukup dengan keyword `forall`.

Hal menarik selanjutnya yaitu deklarasi type `P` di scope yang berbeda dengan type variable `R`, ia muncul satu level di bawahnya. Teknik ini biasa disebut [Rank-N types](https://wiki.haskell.org/Rank-N_types).

Rasa-rasanya mirip film <cite>[Inception](https://www.imdb.com/title/tt1375666/)</cite> tapi pake types. `P` ada di bawah `R`, dan `R` tidak keluar dari `ChunkCPS`, membuatnya _parameterless type_. Tanpa parameter, kita tak lagi punya kewajiban untuk mengisinya dengan type argument.

Mari perbarui type `ChunksMap`.

```ts
type ChunksMap = Map<string, ChunkCPS> // No more type arguments!

const chunks: ChunksMap = new Map()
chunks.set('header',  createChunk([() => Promise.resolve({}), Header]))
chunks.set('profile', createChunk([() => Api.getProfileProps(), Profile]))
chunks.set('sidebar', createChunk([() => Api.getSidebarProps(), Sidebar]))
```

Hasil "expansi kode" di atas beserta _type instantiation_-nya kira-kira berupa:

```ts
chunks.set('header',  (next) => next<{}>([..., Header]))
chunks.set('profile', (next) => next<ProfileProps>([..., Profile]))
chunks.set('sidebar', (next) => next<SideBarProps>([..., Sidebar]))
```

Terus gimana caranya biar value di dalam `ChunksMap` bisa dieksekusi? Kita tahu bahwa value-value ini hanyalah berupa function (sebut saja `unwrap`) yang menerima function lain (`next`) yang akhirnya mengkonsumsi `Chunk<P>`. Sekarang tinggal ikuti type-nya.

```ts {hl_lines=[5]}
chunks.forEach((unwrap, key) => {
  const el = document.getElementById(key)
  if (!el) return

  unwrap(function next(chunk) {
    const [fetchProps, comp] = chunk
    fetchProps().then((props) => {
      ReactDOM.hydrateRoot(el, React.createElement(comp, props))
    })
  })
})
```

Meng-hover kursor di atas `fetchProps` dan `comp` menghasilkan `() => Promise<P>` dan `React.ComponentType<P>`. Kita gak kehilangan type `P`! 🎉

## Kok Bisa CPS?

Nah ini pertanyaan bagus. Gimana ceritanya _existential type_ bisa diekspresikan lewat CPS? Saya coba jawab dengan pengetahuan logic saya yang terbatas. Mari pahami 2 hal terlebih dahulu:

1. _Propositions as types_. Types dapat dilihat sebagai suatu statement yang, jika benar ('true'), memiliki bukti yang direpresentasikan lewat runtime value. Misal `number`, bisa dibuktikan lewat `1`, `2`, `99`, dst. Atau type `string` yang bisa dibuktikan dengan `"any_string"`. Setiap type di Typescript punya representasi runtime value, kecuali type `never`. Ia tak memiliki runtime value. Karenanya, type `never` bersifat 'false'.

2. Menurut ilmu logic, suatu value bisa diungkapkan lewat _double negation_. `type A = Not<Not<A>>`. `true` sama dengan `!!true`

Mari kita ambil contoh type `string`. Ia bersifat 'true' (ada representasi value saat runtime). `Not<string>` seharusnya bersifat 'false', layaknya `never`. `Not<string>` juga dapat dieskpresikan lewat `(str: string) => never` yang kira-kira dibaca, "kalau saya punya sebuah string, saya akan **membuat sesuatu yang mustahil ada**!". Ini sama aja kayak bilang, dengan string kita bisa menghasilkan "bukti" untuk type `never`. Ini kontradiksi, gak boleh terjadi. Oleh sebab itu `(str: string) => never` praktisnya bersifat 'false'.[^1]

Lewat asas ini bisa kita tarik rumus dimana [^2]
1. `Not<A> == <A>(_: A) => never`, dan
2. `Not<Not<A>> == (fn: <A>(_: A) => never) => never`

Balik ke type `Chunk` di bagian sebelumnya, _double negation_ dari `Chunk` adalah `(next: <P>(chunk: Chunk<P>) => never) => never`. Satu masalah besar dengan type ini adalah ia tak berguna: kita cuman dapat `never`, sedangkan kita perlu sesuatu yang konkrit agar komputasi ini bermakna. Lihatlah `Array<T>` yang bisa dicari tahu panjang array-nya, diambil element pertamanya, dll, namun type `T` tetap tidak bisa kita konsumsi secara langsung karena ia abstrak. Dalam hal ini `Array`-lah yang membuat komputasi dengan `T` berguna. Lewat analogi yang sama kita **musti substitusi `never` dengan suatu _quantifier_ (type) agar dapat menghasilkan value yang bisa dikonsumsi**, menjadi `<R>(next: <P>(chunk: Chunk<P>) => R) => R`.[^3] Terlihat familiar?

<details>
  <summary>Kita juga bisa mengaplikasikan double negation ke union type lho!</summary>

  Untuk union `A | B`:
  1. `Not<A | B>` menghasilkan `(<A>(a: A) => R) & (<B>(b: B) => R)`. Bila dieskpresikan dengan tuple menjadi `[<A>(a: A) => R, <B>(b: B) => R]`
  1. `Not<Not<A | B>>` menghasilkan `<R>(fnA: <A>(a: A) => R, fnB: <B>(b: B) => R) => R`
</details>

## Private Type

{{% message info %}}
  Kode pada section ini dapat dilihat di [playground berikut](https://www.typescriptlang.org/play?#code/JYOwLgpgTgZghgYwgAgCpTiAzoswD2IAPKvgNYQgB8yA3gFDLIDml0ckpFIAFAJQAuNOUqNkCABYQEZAEJwANpiQ8wIkEK6VByEAFcAtgCNoYgCYQADvizAwPOAfx7wQ-cegAaZGu6b1Ou4mUOYQRnYOTi5gbobB3r6U-tyBcaYAvvT0YACelijomDgIeIQAwgAKAMrIALzIRABKVDwgEAAeMQ1a1DwA+poY2LgExD1UfHU0jZO101kIhFhgyPIgZFU5cFDAcBJ1yPxTdGKL2CtGisoo9QBsAOx9AAwvzy+nSyv4lliDRSOEIhYHLGfAKGj1BhMJisNoYTjqIRHObITaghQ8ADkcB86kxfE8YiYkmkciuICQSMSIFmNEuSgpEEJ0OQFmsti6kWc4ASAWO9OuyAA1MhHNywMzoRZwpyxdFeSl+eSkMgALSiqI8sSZYmfVn4QrDEqjP5G0ogSo1eo8GA0462njfLB8LJMKAQMB6KAgOj6w3Fc3ITKZehnZbIGCgRTyBmU3RpKAHNYbLY7Pb8AB0ZgNQwDox4jp+tJOuvOuO4BydGdh7AR3H4ruQCg9yAFjMrPwzJJkMeuqgCYjbKvqVbZNgiAEYAKxPN5PBXaQfKm7IUdhSevF7z8uLsTuz3e1vL+jpF2hvX7nYQABuEDMPSTmBT212Ekz2f9AN4hedxyh4j1akOywas2HhCAegbPcPS9H1qRPPggA).
{{% /message %}}

Sekarang saatnya kita eksplor studi kasus lain dimana kita ingin menyembunyikan suatu type dari dunia luar dengan memanfaatkan _existential type_.

```ts
interface Transaction {
  exists Token

  generateToken(): Token
  checkBalance(token: Token): number
  deposit(amount: number, token: Token): number
  debit(amount: number, token: Token): number
}
```

Di sini type `Token` hanya digunakan **di dalam** `Transaction`, gak bocor keluar. Dua hal yang perlu dicatat:
1. Consumer `Transaction` gak tahu menahu soal type ini. "Pokoknya _ada/eksis_ type yang digunakan oleh `Transaction`". Gimana bentuk type-nya, wallahu a'lam.
2. Implementor `Transaction` memiliki kemampuan untuk menentukan concrete type dari `Token` dan punya akses penuh terhadapnya.

Menggunakan CPS, type `Transaction` berubah menjadi

```ts
interface Transaction<Token> {
  generateToken(): Token
  checkBalance(token: Token): number
  deposit(amount: number, token: Token): number
  debit(amount: number, token: Token): number
}

type TransactionCPS = <R>(next: <Token>(_: Transaction<Token>) => R) => R
```

 Mari lihat contoh di bawah ini, `BankSyariah` sebagai implementor `Transaction` menginstansiasi type `Token` dengan `symbol`. Dan `BankRut` lebih memilih UUID yang bertipe `string`.

```ts {hl_lines=["3-4","15-16"]}
const BankSyariah = () => {
  const balance = 67_000_000
  const ops: Transaction<symbol> = {
    generateToken: () => Symbol('a token'),
    checkBalance: (token) => balance,
    deposit: (amount, token) => balance + amount,
    debit: (amount, token) => balance - amount,
  }
  const doTransaction: TransactionCPS = (fn) => fn(ops)

  return { doTransaction }
}

const BankRut = () => {
  const ops: Transaction<string> = {
    generateToken: () => uuidv4(),
    // ...
  }

  // ...
}
```

Berbanding terbalik dengan implementor, pengguna `Transaction` tidak bisa menginspeksi type `Token`. Ia terlihat seperti generic type biasa—tak diketahui instance-nya. Dan sebenarnya gak penting juga untuk diketahui.

```ts {hl_lines=[2]}
const finalBalance: number = BankSyariah().doTransaction((ops) => {
  const token = ops.generateToken()

  let balance = ops.checkBalance(token)
  balance = ops.deposit(150_000, token)
  balance = ops.debit(100_000, token)

  return balance
})
```

Menempatkan kursor di atas `token` pada baris kedua memberikan informasi inference `const token: Token`.

Type `Token` gak bisa kabur keluar dari scope-nya, artinya mengembalikan type `Token` di return statement bakal resolve ke `unknown`. Kenapa gak sekalian error aja saya ndak tahu 🤷.

```ts
// const retrievedToken: unknown
const retrievedToken = BankSyariah().doTransaction((ops) => {
  const token = ops.generateToken()

  return token
})
```

Sebenernya sih ya bisa aja _cheating_ pake semacam `console.log` gitu kan untuk tahu bentuk aslinya. Tapi kalau kamu lagi buat sebuah kontrak dan ada type yang ingin disembunyikan—baik untuk mengurangi type parameter di sisi consumer maupun untuk alasan _correctness_ semata—_existential type_ mungkin bisa jadi awal yang baik.

## Penutup

Perbedaan mendasar antara _universally quantified variable_ (type parameter biasa) dengan _existentially quantified variable_ (well, _existetial type_) adalah:
1. Bila consumer dapat menentukan instance type-nya, maka ia universal.
2. Bila consumer harus menggunakan type yang sudah ditentukan untuknya, maka ia eksistensial.

Dan dari contoh-contoh di atas, baik `P`-nya `Chunk` maupun `Token`-nya `Transaction` consumer tak punya kontrol untuk menginstansiasinya. Karena itu keduanya eksistensial.

Sayangnya bahasa sepopuler Typescript belum sepenuhnya mendukung fitur ini, padahal potensinya besar. Problem yang kelihatannya sederhana jadi butuh solusi yang kompleks: harus ditulis menggunakan continuation-passing style. Semoga kedepannya Typescript segera mengadopsi _existential type_.

[^1]: Selengkapnya bisa dibaca di [Type systems and logic](https://codewords.recurse.com/issues/one/type-systems-and-logic)
[^2]: Tak berlaku di classical logic dimana negasi itu _reversible_. Type system menggunakan constructive logic.
[^3]: https://stackoverflow.com/a/14299983. Penjelasan dengan gambar dapat ditemukan [di sini](https://upload.wikimedia.org/wikiversity/en/8/81/MP3.1D.Mut.Existential.20210904.pdf)
