---
title: "Nominal Types di Typescript"
date: 2023-07-24T13:50:00+02:00
description: "Untuk membedakan dua buah type dengan struktur yang sama"
images: ["/uploads/measurement.jpg"]
image:
  src: "/uploads/measurement.jpg"
  caption: Image by <a href="https://pixabay.com/users/stevepb-282134/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=695723">Steve Buissinne</a> from <a href="https://pixabay.com//?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=695723">Pixabay</a>
tags: ["typescript", "types"]
categories: ["programming", "type system", "typescript"]
draft: false
---

## Structural Typing

Typescript menggunakan pendekatan _structual typing_ ketika melihat kompatibilitas sebuah type, dimana dua buah type dianggap "sama" (atau assignable) bila keduanya memiliki struktur yang sama (umumnya dengan melihat propertinya).

```ts
class User { name: number }
class Person { name: number }

let user = new User()
let person = new Person()

user = person
person = user
```

Baik `user` maupun `person` keduanya bisa di-assign oleh yang lainnya karena sama-sama memiliki property `name`, padahal saat runtime kedua object sebenarnya berbeda.

```ts
user instanceof Person // false
person instanceof User // false
```

Di type system yang nominal ("nomen" dalam bahasa Latin berarti "nama"), dua buah type dianggap berbeda kalau namanya berbeda.

Namun ada beberapa kasus dimana _structual typing_ tak digunakan di TS ketika menilai kompatibilitas satu type dengan yang lainnya, padahal struktur keduanya sama. Salah satu bentuk simple _nominal typing_ di TS adalah enum.

## Enum

Enum bisa bersifat nominal asalkan namanya berbeda, walau strukturnya sama.

```ts
enum Answer { yes, no }
enum Confirm { yes, no }

let answer = Answer.no
let confirm = Confirm.no

╭─────────────────────────────────────────────────────────╮
│Type 'Answer' is not assignable to type 'Confirm'.(2322) │
╰──⌄──────────────────────────────────────────────────────╯
answer = confirm
^^^^^^
╭─────────────────────────────────────────────────────────╮
│Type 'Confirm' is not assignable to type 'Answer'.(2322) │
╰──⌄──────────────────────────────────────────────────────╯
confirm = answer
^^^^^^^
```

Kedua enum sama-sama memiliki `yes` dan `no`, namun menurut TS tidak kompatibel. Hal yang serupa juga bisa diemulasi dengan class yang memiliki private properties.

## Private properties

Dua buah class dianggap tidak kompatibel bila salah satunya memiliki setidaknya satu private property. Satu kasus yang menarik yaitu ketika ingin membedakan `Second` dengan `Millisecond`.

```ts
class Second {
  constructor (private val: number) {}
  get value() { return this.val }
}
class Millisecond {
  constructor (private val: number) {}
  get value() { return this.val }
}

function wait(second: Second) { ... }

╭───────────────────────────────────────────────────────────────────────────────╮
│Argument of type 'Millisecond' is not assignable to parameter of type 'Second'.│
│  Types have separate declarations of a private property 'val'.(2345)          │
╰──────────────⌄────────────────────────────────────────────────────────────────╯
wait(new Millisecond(1000))
     ^^^^^^^^^^^^^^^^^^^^^
wait(new Second(1))
```

Kok bisa? Selengkapnya bisa dibaca di [issue ini](https://github.com/microsoft/TypeScript/issues/18499#issuecomment-329842291). Intinya, class yang memiliki private property tak lagi structural sehingga `Millisecond` dianggap berbeda dengan `Second` walaupun keduanya punya properti `val` dan `value`.

Satu pertanyaan mungkin muncul: apa bisa _nominal typing_ diterapkan juga untuk primitive type? Sebenarnya kan millisecond dan second itu cukup dieskpresikan dengan `number`, apa bisa TS membedakan dua buah `number`?

Enter branding.

## Branding

Branding menggabungkan type dasar (dalam hal ini `number`) dengan type lain yang bisa dibedakan oleh type checker melalui intersection.

```ts
type Second = number & { __brand: 'Second' }
type Millisecond = number & { __brand: 'Millisecond' }

function wait(second: Second) { ... }

╭───────────────────────────────────────────────────────────────────────────────╮
│Argument of type 'Millisecond' is not assignable to parameter of type 'Second'.│
│  Type 'Millisecond' is not assignable to type '{ __brand: "Second"; }'.       │
│    Types of property '__brand' are incompatible.                              │
│      Type '"Millisecond"' is not assignable to type '"Second"'.(2345)         │
╰──────────────⌄────────────────────────────────────────────────────────────────╯
wait(1000 as Millisecond)
     ^^^^^^^^^^^^^^^^^^^
╭──────────────────────────────────────────────────────────────────────────╮
│Argument of type 'number' is not assignable to parameter of type 'Second'.│
│  Type 'number' is not assignable to type '{ __brand: "Second"; }'.(2345) │
╰────⌄─────────────────────────────────────────────────────────────────────╯
wait(1)
     ^
wait(1 as Second) // typechecks ✅
```

Cara ini diterapkan juga di [source code TS sendiri](https://github.com/microsoft/TypeScript/blob/01b18215eccec4e3b32743ab545bf8c6b570d782/src/compiler/types.ts#L24) dan [di test case mereka](https://github.com/microsoft/TypeScript/blob/01b18215eccec4e3b32743ab545bf8c6b570d782/tests/cases/conformance/types/intersection/intersectionAsWeakTypeSource.ts#L15).

Dengan branding, setiap kali consumer ingin memanggil `wait`, mereka harus secara eksplisit cast dari number ke `Second`. Lumayan untuk nambah satu layer safety, dimana mereka sadar dan tahu betul apa yang mereka lakukan.

Bila ada satu kekurangan dari cara ini, ialah properti `__brand` yang masih bisa diakses dan muncul di autocomplete: `oneSecond.__brand`, padahal properti ini imaginary, ia tak benar-benar ada saat runtime.

Ngakalinnya bisa pake empty string.

```ts
type Second = number & { '': 'Second' }
type Millisecond = number & { '': 'Millisecond' }
```

Gak pernah kan liat kode `oneSecond['']`? 😁

Oh ya, cara ini juga berasumsi bahwa programmer mengisi properti brand-nya dengan string yang unik. In most cases it's fine! Paling-paling agak jadi PR kalau nulis library, karena bisa aja bentrok dengan library lain yang sama-sama pake _nominal types_. Tapi ini skenario hipotetikal aja!

## Unique symbol

Tekniknya sama dengan branding, namun gak butuh string unik untuk membuat suatu type nominal. Yaitu pake `unique symbol`!

```ts
type Second = number & { readonly '': unique symbol }
type Millisecond = number & { readonly '': unique symbol }
```

Kita tahu `unique symbol` itu type dari `Symbol()`, dimana satu symbol dengan symbol lainnya sudah pasti unik. Nah keunikan ini dibawa ke type level!

Alangkah lebih asyik bila TS juga support penulisan type di bawah ini, untuk mengurangi repetitiveness.

```ts
type Nominal<T> = T & { readonly '': unique symbol }

type Second = Nominal<number>
type Millisecond = Nominal<number>
```

Namun sayangnya cara ini belum manjur karena keunikan symbol di type `Nominal` dipake bersama-sama oleh consumer :( Jadi kode di bawah ini typecheck:

```ts
wait(1000 as Millisecond) // seharusnya error!
```

## Penggunaan lain

Penggunaan _nominal types_ ini banyak aplikasinya, diantaranya:
- Unit:
  - `Kilometer`, `Meter`, `Milimeter`
  - `Celcius`, `Fahrenheit`, `Kevin`
- Mata uang: `USD`, `EUR`, `IDR`
- ID: `PostID`, `CommentID`
- String: `Email`, `Username`, `Password`
- Non-empty: `NonEmptyArray`, `NonEmptyString`
- Range:
  - `Discount`, hanya valid bila 0 < discount < 100
  - `PostiveInteger`, dimana 0 < angka
  - `NegativeInteger`, dimana angka < 0

Type guard bisa dimanfaatkan untuk tipe data yang membutuhkan validasi.

```ts
function isDiscount(dicount: number): discount is Discount {
  return 0 < discount && discount < 100
}

function applyDiscount (price: number, discount: Discount) { ... }

const discount = 50
if (isDiscount(discount)) {
  applyDiscount(12_000, discount)
}
```

Pengalaman pribadi: saban hari saya hampir pernah menampilkan diskon yang negatif karena tak adanya cek saat compile time 🐞

## Kesimpulan

Manfaatkan _nominal types_ bila ingin membedakan satu type dari yang lainnya dengan struktur yang sama supaya program lebih type safe. Tak akan ada parameter yang tertukar hanya karena type-nya mirip-mirip.

Semoga artikel ini bermanfaat!

<script>
  let loaded = false
  window.requestAnimationFrame(function waitForPrism () {
    if (loaded) { return }

    if (window.Prism === undefined) {
      return window.requestAnimationFrame(waitForPrism)
    }

    window.setTimeout(function () {
      $('.line').each((i, el) => {
        if ($(el).find('.err').length > 0) {
          $(el).addClass('balloon')
        }
      })
    }, 0)

    loaded = true
  })
</script>

<style>
  .line.balloon {
    color: #999;
  }
  .line.balloon .token {
    color: inherit !important;
  }
</style>
