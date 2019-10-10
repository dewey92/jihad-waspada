---
title: "Kenalan Dulu sama Type Class"
date: 2019-10-10T20:39:43+02:00
description: "Type class ini agak tricky dijelaskan untuk beginner. Mirip-mirip kayak interface, tapi ya bukan kayak interface. Terus gimana dong?"
images: ["/uploads/bandage.jpg"]
image:
  src: "/uploads/bandage.jpg"
  caption: Image by <a href="https://pixabay.com/users/HeungSoon-4523762/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2671511">HeungSoon</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2671511">Pixabay</a>
tags: ["purescript", "types"]
categories: ["programming", "purescript", "type system"]
draft: false
---

Tidak semua bahasa pemrograman mendukung fitur type class. Beberapa orang bilang konsep type class _identik_ dengan konsep interface pada bahasa-bahasa seperti Java, C#, atau Typescript, dimana type class dan interface sama-sama bertujuan untuk menyediakan function yang polymorphic, walaupun sebenarnya keduanya tidak 100% sama (hence identik).

Saya akan menggunakan Purescript untuk menjelaskan konsep type class pada article ini.

## Appendabale

Ambil String dan Array. Keduanya memiliki sifat yang sama yaitu dapat **digabungkan**; string dengan string, array dengan array. Di Javascript, penggabungan ini bisa dicapai dengan menggunakan operator `+` atau `concat`.

```js
λ> 'jihad ' + 'waspada'
'jihad waspada'

λ> [1, 2] + [3]
[1, 2, 3]
```

Konsep type class bisa digunakan untuk meng-capture kesamaan behaviour antara String dan Array. Mari kita namakan type class ini `Appendable`.

```hs
-- Type class dalam Purescript ditulis dengan keyword `class`.
-- Jangan samakan `class` ini dengan istilah class yang ada
-- di bahasa-bahasa OOP pada umumnya
class Appendable a where
  append :: a -> a -> a
```

Disini `Appendable` adalah nama class-nya, dan `append` adalah method tunggalnya. Type parameter `a` nanti akan diisi oleh type yang ingin memiliki instance `Appendable` (untuk saat ini String dan Array). Type class pada umumnya hanyalah kumpulan-kumpulan method tanpa implementasi, hanya type signature, maka gak heran sebagian menyandingkannya dengan konsep interface di Java atau bahasa sejenis.

Agar function `append` tersebut dapat dipanggil, kita musti buat implementasi atau instance dari class `Appendable`.

```hs
instance appendableStr :: Appendable String where
  append x y = ... -- implementation details

instance appendableArr :: Appendable (Array xs) where
  append x y = ... -- implementation details
```

Potongan kode di atas mengimplikasikan bahwa String dan Array kini memiliki instance `Appendable`, terlihat dari type parameter `a` pada class `Appendable a` yang sekarang sudah terisi (tersubstitusi) dengan String/Array. Oh ya, detil implementasinya bisa berupa _apapun_, namun kali ini saya biarkan kosong saja.

Kita sudah define instance-nya, dan `append` akhirnya bisa digunakan oleh String dan Array 🥳

```hs
λ> append "jihad " "waspada"
"jihad waspada"

λ> append [1, 2] [3]
[1, 2, 3]
```

Bagaimana kalau `append` diisi dengan type yang belum memiliki instance `Appendable` seperti let's say, Int? Pasti error atuh..

```hs
λ> append 11 99
-- ERROR!
-- No type class instance was found for `Appendable Int`
```

Walaupun type class sekilas _terlihat_ seperti interface, ada perbedaan yang sangat mendasar, yaitu type class memungkinkan kita untuk membuat instance untuk type yang bukan milik kita. Lihat saja String dan Array, keduanya adalah **tipe data primitif** Purescript, bukan buatan kita. Namun kita tetap bisa memberikan instance (dalam hal ini instance `Appendable`) kepada mereka. Kemampuan "memberikan instance" pada suatu tipe data yang **tidak kita buat sendiri** (i.e String dan Array) ini biasanya hampir tidak bisa dilakukan oleh interface.

Agar lebih jelas, misal ada third-party library bernama `Color` dan kita memiliki interface kita sendiri, `Printable`.

```ts
interface Printable {
  print: () => string;
}
```

Bagaimana caranya agar `Color` tersebut meng-implement interface `Printable`? Gak bisa 😢. Mau gak mau harus kita wrap dengan class lain.

```ts
import Color from 'thirdPartyLib'

class MyColor implements Printable {
  constructor (private color: Color) {}

  print = () => {
    if (this.color.isRed) return 'RED';
    if (this.color.isBlue) return 'BLUE';
    return 'GREEN'
  }
}
```

Yuck!

Dengan type class, kita dengan mudahnya bisa memberikan behaviour tambahan pada `Color`:

```hs
import ThirdPartyLib (Color)

class Printable a where
  print :: a -> String

instance printableColor :: Printable Color where
  print x =
    if x.isRead then "RED" else
    if x.isBlue then "BLUE" else
    "GREEN"

-- Harusnya pake ADT. Tapi yaa.. contoh aja
```
---

### INTERMEZZO

> 📝 Di Typescript, kemampuan ini "bisa" dicapai dengan memanfaatkan fitur [augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#global-augmentation) dan JS prototypes!

```ts
// Buat interface dasarnya
interface Appendable<A> {
  append: (other: A) => A
}

// interface merging!
interface String extends Appendable<string> { }
interface Array<T> extends Appendable<Array<T>> { }

// Tambah behaviour dengan prototype
String.prototype.append = function (other) {
  return this + other;
}
Array.prototype.append = function (other) {
  return this.concat(other); // atau `this + other`
};

console.log('jihad '.append('waspada')) // => 'jihad waspada'
console.log([1, 2].append([3])) // => [1, 2, 3]
```

---

## Subtyping
Sama seperti interface, type class juga memiliki konsep subtyping dimana sebuah type class dapat "meng-extend" method dari type class lainnya.

```hs
class Appendable a <= HazDefault a where
  defaultVal :: a
```

Di sini class `HazDefault` dapat dianggap "meng-extend" class `Appendable`. Namun lagi, saya menolak untuk menerima anggapan ini sepenuhnya 😄. Alih-alih menggunakan keyword "extend" atau "subtyping", saya lebih suka dengan keyword **constraining**. Sehingga notasi class di atas dapat dibaca dengan: "suatu data dapat memiliki instance `HazDefault` **asalkan** ia memiliki instance `Appendable`".

Bahasa kerennya: `HazDefault` is **constrained** by `Appendable`.

String dan Array sudah memiliki instance `Appendable`, maka sah-sah saja bagi mereka untuk juga memiliki instance `HazDefault` 😉

```hs
instance hazDefaultStr :: HazDefault String where
  defaultVal = ""

instance hazDefaultArr :: HazDefault (Array xs) where
  defaultVal = []
--

λ> append "jihad" defaultVal
"jihad"
λ> append [1, 2] defaultVal
[1, 2]
```

Compiler akan complain dengan pesan yang cukup jelas kalo kita iseng membuat instance `HazDefault` untuk tipe data yang belum memiliki instance `Appendable`.

```hs
instance hazDefaultInt :: HazDefault Int where
  defaultVal = 0

-- ERROR!
-- No type class instance was found for `Appendable Int`
```

## Constraint
Type class is all about instance and constraint. Mari perhatikan contoh berikut:

```hs
guard :: ∀ a. HazDefault a => Boolean -> a -> a
guard true val = val
guard false _ = defaultVal
```

Constraint type class pada sebuah function dipisahkan dengan symbol `=>`. Fungsi `guard` menerima dua buah argument; `Boolean` dan `a`, dan mengembalikan `a`. Namun `a` di sini tidak boleh sembarang type, ia harus mempunyai instance `HazDefault`.

In fact, kalau kita menginspeksi type `append` dan `defaultVal` di REPL, yang kita lihat sebenarnya adalah:

```hs
λ> :t append
append :: Appendable a => a -> a -> a

λ> :t defaultVal
defaultVal :: HazDefault a => a
```

Tidak mengejutkan 😄

Balik ke pembahasan utama. Kita sudah tahu bahwa fungsi `guard` ter-contraint dengan class `HazDefault` pada type parameter `a`, yang berarti `a` hanya boleh diisi dengan String atau Array.

{{< highlight hs "hl_lines=7 10" >}}
type User = String

isRoot :: User -> Boolean
isRoot user = user == "jihad"

welcomeMessage :: User -> String
welcomeMessage user = guard (isRoot user) "Welcome, root!"

access :: User -> Array Int
access user = guard (isRoot user) [7, 7, 7]
--

λ> welcomeMessage "jihad"
"Welcome, root!"
λ> welcomeMessage "not admin"
""

λ> access "jihad"
[7, 7, 7]
λ> access "not admin"
[]
{{< /highlight >}}

Lagi, compiler akan menolak untuk memberikan lampu hijau jika function `guard` dipanggil dengan tipe data yang tidak memiliki instance `HazDefault` seperti Int.

```hs
invalid = guard true 123
          ^^^^^^^^^^^^^^

-- ERROR!
-- No type class instance was found for `HazDefault Int`
```

## Methods Injection
Selain kemampuan membatasi akses pada suatu function, constraint type class pada dasarnya **memberikan semua method-nya** secara implisit. Untuk lebih jelasnya, mari modifikasi function `guard` barusan.

```hs
guard :: ∀ a. HazDefault a => Boolean -> a -> a -> a
guard true x y = x `append` y
guard false _ _ = defaultVal
```

Dengan adanya constraint `HazDefault` di type signature, function `guard` memiliki izin untuk mengakses method-method yang ada pada class `HazDefault` (`append` dan `defaultVal`). Under the hood, compiler melakukan proses desugaring seperti berikut.

```hs
guard :: ∀ a. HazDefaultDict a -> Boolean -> a -> a -> a
guard { append, defaultVal } true x y = x `append` y
guard { append, defaultVal } false _ _ = defaultVal
```

> Proses bagaimana compiler melakukan desugaring dan memberikan semua method-nya secara implisit dapat dibaca di artikel saya tentang [Type Class Dan Cara Kerjanya Di Balik Layar]({{< ref "/post/type-class-dan-cara-kerjanya-di-balik-layar.md" >}}).

Dengan kata lain, function `append` dan `defaultVal` bersifat eksklusif, tidak bisa sembarang dipanggil oleh function lain. **Caller harus memberikan constraint di type signature-nya**. Jika tidak, compiler akan complain.

```hs
invalid :: ∀ a. a -> a -> a
invalid x y = x `append` y

-- ERROR!
-- No type class instance was found for `Appendable a`
```

## Readability dan Testability

Bicara agak real world, type class juga dapat membantu readability ketika ada suatu function yang melakukan side effect.

```hs
class Monad m <= MonadCache m where
  readCache :: Path -> m (Maybe String)
  writeCache :: String -> Path -> m Unit

class Monad m <= MonadUserDb m where
  getUser :: UserId -> m (Maybe User)

fetchUser :: ∀ m.
  MonadCache m =>
  MonadUserDb m =>
  UserId -> m (Maybe User)
fetchUser uId = do
  cache <- readCache ("/cache/user" <> uId)
  ...
  ...
```

Fungsi `fetcUser` ter-constraint dengan dua buah type class: `MonadCache` dan `MonadUserDb`, yang memungkinkan untuk memanggil fungsi `readCache`, `writeCache`, dan `getUser` di dalamnya. Fungsi `fetchUser` juga secara tidak langsung ter-constraint dengan type class `Monad` sehingga kita bisa langsung menggunakan`do` binding 😍

Gak hanya itu, kita juga bisa menyimpulkan dari melihat type signature-nya saja bahwa `fetchUser` bakal berinteraksi dengan cache dan database (sebagai side effect) tanpa terikat dengan implementasi apapun! Implementasi tergantung konteks caller-nya. Misal ketika testing, instance bisa dibuat semau kita. Style ini biasa disebut dengan MTL/Tagless Final.

{{< highlight hs "hl_lines=12-14 16-17" >}}
newtype TestM a = TesM (Aff a)

runTest :: TestM ~> Aff
runTest (TestM a) = a

-- derive instances
derive newtype instance functorTestM :: Functor TestM
...
...
derive newtype instance monadTestM :: Monad TestM

instance monadCacheTestM :: MonadCache TestM where
  readCache _ = pure $ Just "user-from-cache"
  writeCache _ _ = pure unit

instance monadUserDbTestM :: MonadUserDb TestM where
  getUser _ = pure $ Just (User "jihad")

it "fetches a user from cache" do
  fromCache <- runTest $ fetchUser 1
  fromCache `shoudlEqual` (Just (User "user-from-cache"))
{{< /highlight >}}

Mungkin pembahasan MTL (Monad Transformer Library) selengkapnya bisa diulas di lain waktu. Namun yang ingin saya tekankan adalah penggunaan type class itu sudah dimana-dimana. `Eq`, `Show`, `Functor`, `Monad`, `Applicative`, `Semigroup` (Appendable), `Monoid` (HazDefault), dan `Traversable` adalah beberapa type class dasar yang wajib dipahami.

Dari type class jugalah kita sebenarnya bisa melihat bahwa pattern dalam programming dapat di-abstraksi sejauh mungkin. Appendable (biasa disebut `Semigroup`) dan HazDefault (biasa disebut `Monoid`) hanyalah contoh kecil saja. Video di bawah ini gak pernah bosen saya rekomendasikan untuk melihat bagaimana cara mengeneralisasi pattern dari sebuah masalah yang berujung pada terbentuknya type class.

{{< youtube GlUcCPmH8wI >}}

<br />
Semoga bermanfaat 🙂