---
title: "Mudahkan Perkerjaanmu dengan Beberapa Tips Typescript Berikut"
date: 2021-05-24T09:58:12+02:00
description: "Tiga tips receh di Typescript"
images: ["/uploads/magic.jpg"]
image:
  src: "/uploads/magic.jpg"
  caption: Image by <a href="https://pixabay.com/users/adinavoicu-485024/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1339696">Adina Voicu</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1339696">Pixabay</a>
tags: ["types", "typescript"]
categories: ["typescript", "programming", "type system"]
draft: false
---

## Menyimpan Hasil Komputasi ke dalam Type Variable

```ts
// NOPE ❌
type Something<T> =
  | Array<Some<Fancy<T>, Or<Long<T>>>>
  | Promise<Some<Fancy<T>, Or<Long<T>>>>
  | { wrapped: Some<Fancy<T>, Or<Long<T>>> }
```

```ts
// YES ✅
type Something<T> = Some<Fancy<T>, Or<Long<T>>> extends infer U
  ? Array<U> | Promise<U> | { wrapped: U }
  : never
```

### Penjelasan

Kadangkala kita harus melakukan manipulasi type yang cukup kompleks dan panjang. Tapi gak asyik kalo type yang compleks tadi harus muncul di beberapa tempat: kita musti copy-paste. Gak DRY banget bro. Bakal jadi asyik kalo Typescript menyediakan cara untuk menyimpan hasil sebuah manupulasi ke dalam sebuah variable lalu tinggal kita panggil variable tersebut ketika dibutuhkan.

Dan kita bisa melakukannya dengan formula `... extends infer TypeVar ? ... : never`. Keyword `infer` behaves seperti pattern matching sebuah type. Dalam hal ini, type yang di-pattern match adalah keseluruhan type, jadi yang masuk ke dalam type variable `U` adalah type yang ada di sebelah kiri kata kunci _extends_.

## Looping Union Type

```ts
// NOPE ❌
type Arrayify_Bad<T> = Array<T>
```

```ts
// YES ✅
type Arrayify_Good<T> = T extends unknown ? Array<T> : never
```

### Penjelasan

Anggap aja kita punya sebuah union `string | number`. Dan kita ingin sebuah type yang bisa mengubahnya menjadi `string[] | number[]`. Jika kita gunakan `Arrayify_Bad<string | number>`, hasilnya bakalan jadi `Array<string | number>`, _not something that we want_. `Arrayify_Good<string | number>` justru menghasilkan `string[] | number[]`. Kenapa?

Karena generic type menjadi **distributif** ketika di-apply dengan conditional type. Bisa dibaca di [dokumentasinya](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#distributive-conditional-types) untuk Penjelasan lebih lanjut.

Sebaliknya, untuk mencegah union type terdistribusi, kita harus gunakan kurung kotak (tuple) di kedua sisi:

```ts
type Arrayify_NonDist<T> = [T] extends [unknown] ? Array<T> : never

type Result = Arrayify_NonDist<string | number>
// => Array<string | number>
```

## Hilangkan Type Alias dengan Mapped Type

```ts
type Human = { name: string, age: number, title: string }
type Cat   = { name: string, age: number, meow: boolean }

type Result = XOR<Human | Cat>
// => Maunya { title: string } | { meow: boolean }

type Keyof<T> = ...
type XAND<T> = ...

/**
 * Get non-overlapping properties in a union type (the dual of XAND)
 */
type XOR<T> = keyof XAND<T> extends infer K
  ? T extends unknown
    ? Omit<T, K & string>
    : never
  : never
```

### Penjelasan

`Keyof` dan `XAND` di atas saya singkat saja karena kurang relevan. `XOR` pun jangan diambil pusing. Poinnya ada di reporting ketika hover variable `Result`:

![Cryptic reporting](/uploads/cryptic-reporting.png)

Menurut saya pribadi, keyword `Omit` dan banyaknya symbol `|` yang muncul sangat mengganggu dan bikin dahi mengernyit. Saya ingin hasil yang lebih _straightforward_. Bayangkan jika bukan hanya `Omit` yang ada di situ, tapi ada custom type lain (i.e dari 3rd party library) yang musti kita telusuri satu-satu. Bikin pusing.

Nah kita bisa _expand_ type alias tadi untuk hasil inspeksi yang lebih _readable_ dengan [mapped type](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html).

```ts
type Result = XOR<Human | Cat> extends infer T
  ? { [K in keyof T]: T[K] } // ⬅️ Simple mapped type
  : never
```

Ketika di-hover, hasil yang kita dapat:

![Better reporting](/uploads/better-reporting.png)

_Much better_!

## Kesimpulan

Saya harap tips-tips kecil di atas dapat membantu teman-teman untuk lebih enjoy memanipulasi type dengan Typescript. _Stay well_!
