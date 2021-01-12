---
title: "Kenalan Dulu sama Type Class"
date: 2019-10-10T20:39:43+02:00
description: "Ad-hoc polymorphism \"interface\""
images: ["/uploads/bandage.jpg"]
image:
  src: "/uploads/bandage.jpg"
  caption: Image by <a href="https://pixabay.com/users/HeungSoon-4523762/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2671511">HeungSoon</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2671511">Pixabay</a>
tags: ["purescript", "types"]
categories: ["programming", "purescript", "type system"]
draft: false
---

Tidak semua bahasa pemrograman mendukung fitur type class. Beberapa orang bilang konsep type class _identik_ dengan konsep interface pada bahasa-bahasa seperti Java, C#, atau Typescript, dimana type class dan interface sama-sama bertujuan untuk menyediakan function yang polymorphic, walaupun sebenarnya keduanya tidak 100% sama. Identik saja.

Saya gunakan Purescript untuk menjelaskan konsep type class pada article ini.

## Give Me the Code

```hs
class Show a where -- [1]
  show :: a -> String  -- [2]
```

1. Kita membuat sebuah type class dengan nama `Show`. Umumnya type class menyediakan satu (atau lebih) type variable yang bakal digunakan di method-nya. Di sini type variable tersebut adalah `a`.
2. Type class `SHow` ini memiliki satu method bernama `show`, yang menerima `a` dan mengembalikan String.

Tidak ada concrete code di sini, tugas type class hanya mendefinisikan struktur (type signature) dari method-method yang bisa digunakan oleh suatu data structure. Seperti interface, hanya kontrak. Detil implementasi diserahkan ke implementornya.

```hs
newtype Email = Email String

instance showEmail :: Show Email where -- [3]
  show (Email e) = "Email: " <> e -- [4]

λ> show (Email "dewey@email.com")
"Email: dewey@email.com"
```

3. Di sini kita membuat instance `Show` untuk `Email` dengan nama `showEmail` (nama instance bisa kita abaikan, tidak terlalu penting untuk saat ini).
4. Implementasi `show`. Inilah gunanya type variable `a` di atas tadi. Sekarang `a` telah di-instantiate dengan `Email` sehingga bisa kita konsumsi di function argument.

Karena perannya yang mirip dengan interface, kita juga bisa membuat instance untuk data structure lain. Sehingga function `show` tidak hanya applicable untuk type Email, tapi juga Password.

```hs
newtype Password = Password String

instance showPassword :: Show Password where
  show _ = "<secret>"

λ> show (Password "SomePassword789_+*!@#")
"<secret>"
```

Kalau type class identik dengan interface, lalu dimana bedanya?

## Type Class vs Interface

Walaupun type class sekilas terlihat seperti interface, ada perbedaan yang sangat mendasar, yaitu type class memungkinkan kita untuk membuat instance terhadap type yang **bukan milik kita**. Oleh karenanya banyak orang yang menyebut type class sebagai "the true ad-hoc polymorphism".

Tak perlu jauh-jauh memikirkan tipe data dari third-party library, kita bahkan bisa memberi instance untuk tipe data primitif seperti Boolean.

```hs
instance showBoolean :: Show Boolean where
  show true = "true"
  show false = "false"
```

Tipe data primitif lainnya seperti Int, String, Array sudah "diurus" oleh [Prelude](https://github.com/purescript/purescript-prelude/blob/a96663b34364fdd0885a200955e35b99f4e58c43/src/Data/Show.purs#L20-L42). Semua di level library, no magic.

## Appendable

Let's dig deeper.

Ambil String dan Array. Keduanya memiliki sifat yang sama yaitu dapat **digabungkan**; string dengan string, array dengan array. Di Javascript, penggabungan ini bisa dicapai dengan menggunakan operator `+` atau `concat`.

```js
λ> 'jihad ' + 'waspada'
'jihad waspada'

λ> [1, 2] + [3]
[1, 2, 3]
```

Behavior atau sifat "bisa digabungkan" ini bisa kita capture dengan sebuah type class, sebut saja `Appendable`.

```hs
class Appendable a where
  append :: a -> a -> a

instance appendableStr :: Appendable String where
  append x y = ... -- implementation details

instance appendableArr :: Appendable (Array xs) where
  append x y = ... -- implementation details

λ> append "jihad " "waspada"
"jihad waspada"

λ> append [1, 2] [3]
[1, 2, 3]
```

Kita baru saja memberikan String dan Array kemampuan untuk bisa digabungkan dengan membuat instance `Appendable`. Detil implementasinya kita biarkan kosong agar contoh code-nya tetap sederhana. Bila kita panggil fungsi `append` dengan tipe data yang belum memiliki instance `Appendable` seperti Int, program akan error.

```hs
λ> append 11 99
-- ERROR!
-- No type class instance was found for `Appendable Int`
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
  return this.concat(other);
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

Type class `HazDefault` di-construct dengan _superclass_ `Appendable`, yang memungkinkan instance `HazDefault` untuk tetap bisa memanggil fungsi `append`, dengan syarat setiap instance `HazDefault` harus juga memiliki instance `Appendable`. Relasi sebuah type class dengan superclass-nya ditandai dengan symbol `<=`.

String dan Array sudah memiliki instance `Appendable`, maka sah-sah saja bagi mereka untuk juga memiliki instance `HazDefault` 😉

```hs
instance hazDefaultStr :: HazDefault String where
  defaultVal = ""

instance hazDefaultArr :: HazDefault (Array xs) where
  defaultVal = []

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

## Overlapping Instances

Sejauh ini kita sudah belajar bagaimana type class berguna untuk memberikan "kemampuan" pada suatu  tipe data. Kita sudah memberi String dan Array "kemampuan" untuk melakukan penggabungan dengan fungsi `append` dan mengembalikan nilai kosongnya dengan fungsi `defaultVal`.

Apa yang terjadi jika kita memberikan kemampuan yang sama dua kali?

```hs
instance hazDefaultStr :: HazDefault String where
  defaultVal = ""

instance hazDefaultStr2 :: HazDefault String where
  defaultVal = "zzz"

-- Overlapping type class instances found for
--
--   HazDefault String
--
-- The following instances were found:
--
--   hazDefailtStr
--   hazDefailtStr2
```

Compiler komplain. Dan sudah bisa ditebak pasti karena kebingungan memilih harus pakai instance yang mana. Kondisi ini disebut Overlapping Instances. Namun jika tetep kekeuh ingin menuliskan overlapping instances, Purescript menyediakan fitur [Instance Chains](https://github.com/purescript/documentation/blob/master/language/Type-Classes.md#instance-chains) yang tidak akan saya bahas di artikel ini.

Tapi kadangkala ada aja kasus dimana suatu data bisa memiliki dua behavior: misal untuk tipe `Int` jika mengimplementasi class `HazDefault`. Nilai default Integer bernilai 0 ketika dijalankan dalam konteks penjumlahan, namun bernilai 1 dalam konteks perkalian. Ketika dihadapkan dengan situasi seperti ini, salah satu cara untuk mengakalinya bisa dengan membungkusnya dengan `newtype`.

```hs
newtype Sum = Sum Int
newtype Prod = Prod Int

instance appendableSum :: Appendable Sum where
  append (Sum a) (Sum b) = Sum (a + b)

instance appendableProd :: Appendable Prod where
  append (Prod a) (Prod b) = Prod (a * b)

instance hazDefaultSum :: HazDefault Sum where
  defaultVal = Sum 0

instance hazDefaultProd :: HazDefault Prod where
  defaultVal = Prod 1
--

λ> append (Sum 99) (Sum 1)
Sum 100
λ> append (Prod 50) (Prod 2)
Prod 100
λ> append (Sum 99) defaultVal
Sum 99
λ> append (Prod 50) defaultVal
Prod 50
```

## Constraint
Type class is all about instance and constraint. Mari perhatikan contoh berikut:

```hs
guard :: ∀ a. HazDefault a => Boolean -> a -> a
guard true val = val
guard false _ = defaultVal
```

Constraint type class pada sebuah function dipisahkan dengan symbol `=>`. Fungsi `guard` menerima dua buah argument; `Boolean` dan `a`, dan mengembalikan `a`. Namun `a` di sini tidak boleh sembarang type, ia harus mempunyai instance `HazDefault`.

Kita sudah tahu bahwa fungsi `guard` ter-contraint dengan class `HazDefault` pada type parameter `a`, yang berarti `a` hanya boleh diisi dengan String atau Array.

```hs {hl_lines=[8,12]}
type User = String

isRoot :: User -> Boolean
isRoot user = user == "jihad"

-- Untuk String
welcomeMessage :: User -> String
welcomeMessage user = guard (isRoot user) "Welcome, root!"

-- Untuk Array
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
```

In fact, kalau kita menginspeksi type `append` dan `defaultVal` di REPL, yang kita lihat sebenarnya adalah:

```hs
λ> :t append
append :: Appendable a => a -> a -> a

λ> :t defaultVal
defaultVal :: HazDefault a => a
```

Tidak mengejutkan 🙂

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

Bicara agak real world, type class juga dapat membantu readability ketika kita ingin melacak side effect apa saja yang bakal dijalankan di sebuah function.

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

Fungsi `fetcUser` ter-constraint dengan dua buah type class: `MonadCache` dan `MonadUserDb`, yang memungkinkan untuk memanggil fungsi `readCache`, `writeCache`, dan `getUser` di dalamnya. Fungsi `fetchUser` juga secara tidak langsung ter-constraint dengan type class `Monad` sehingga kita bisa langsung menggunakan `do` binding.

Gak hanya itu, kita juga bisa menyimpulkan dari melihat type signature-nya saja bahwa `fetchUser` bakal berinteraksi dengan cache dan database (side effect) tanpa terikat dengan implementasi apapun. Implementasi tergantung konteks caller-nya. Misal ketika testing, instance bisa dibuat semau kita.

```hs {hl_lines=["6-8","10-11"]}
newtype TestM a = TesM (Aff a)

runTest :: TestM ~> Aff
runTest (TestM a) = a

instance monadCacheTestM :: MonadCache TestM where
  readCache _ = pure $ Just "user-from-cache"
  writeCache _ _ = pure unit

instance monadUserDbTestM :: MonadUserDb TestM where
  getUser _ = pure $ Just (User "jihad")

it "fetches a user from cache" do
  fromCache <- runTest $ fetchUser 1
  fromCache `shoudlEqual` (Just (User "user-from-cache"))
```

## Penutup

Penggunaan type class ada dimana-dimana. `Eq`, `Show`, `Functor`, `Monad`, `Applicative`, `Semigroup` (Appendable), `Monoid` (HazDefault), dan `Traversable` adalah beberapa type class dasar yang wajib dipahami.

Dari type class jugalah kita sebenarnya bisa melihat bahwa pattern dalam programming dapat di-abstraksi sejauh mungkin. Appendable (biasa disebut `Semigroup`) dan HazDefault (biasa disebut `Monoid`) hanyalah contoh kecil saja. Video di bawah ini gak pernah bosen saya rekomendasikan untuk melihat bagaimana cara mengeneralisasi pattern dari sebuah masalah yang berujung pada terbentuknya type class.

{{< youtube GlUcCPmH8wI >}}

<br />
Semoga bermanfaat 🙂