---
title: "Akses Global Values dengan Reader Monad"
date: 2020-03-21T12:46:00+01:00
description: "Reader Monad sebagai wadah penyimpanan global values"
images: ["/uploads/cup.jpg"]
image:
  src: "/uploads/cup.jpg"
  caption: Image by <a href="https://pixabay.com/users/JillWellington-334088/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1975215">Jill Wellington</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1975215">Pixabay</a>
tags: ["purescript", "haskell", "types", "monad", "functionalprogramming"]
categories: ["programming", "purescript", "functional programming"]
draft: false
---

Sampai beberapa tahun yang lalu (dan mungkin masih sampai sekarang), komunitas React sempat dibuat hype dengan adanya Redux sebagai _state management library_. Walaupun pada kenyataannya banyak juga yang hanya menggunakan Redux sebagai wadah untuk _global/shared state_ mereka, dalihnya sih agar terhindar dari _props drilling_ atau menyuplai _props_ secara eksplisit ke setiap level component tree.

Lalu muncul React Context, yang tujuan utamanya persis seperti yang barusan: agar developer terhindar dari _props drilling_. Dikutip dari dokumentasi resmi React:

> Context provides a way to share values like these between components without having to explicitly pass a prop through every level of the tree.

Artikel ini saya tulis lebih karena teringatnya saya pada Reader Monad sebagai salah satu cara untuk menyimpan dan mengatur _global values_ agar deep-nested functions tetap punya akses tanpa _props drilling_. Persis seperti React Context (bedanya Reader type-safe). Dan saya akan menjelaskannya dengan Purescript.

## Apa itu Reader Monad

Seperti yang telah dijelaskan sebelumnya, Reader Monad berfungsi sebagai wadah penyimpanan suatu shared value. Shared value tersebut bisa berupa object `connection` yang nantinya dibutuhkan oleh setiap function yang menjalankan database query. Bisa juga berupa kumpulan nilai environment (configuration) yang biasa "disimpan" di file `.env`.

Definisi Reader Monad sangat simple:

```hs
Reader env a
```

dimana `env` adalah shared value kita dan `a` adalah nilai yang dihasilkan dari shared value tersebut, mirip sebuah function `env -> a`. Dan memang sejatinya Reader Monad adalah function `env -> a` 😄. Tapi pembahasan function sebagai Reader kita kesampingkan dulu.

Karena Reader ([memiliki instance Monad](https://github.com/purescript/purescript-transformers/blob/0e473e5ef0e294615ca0d9aab0bcffee47b2870d/src/Control/Monad/Reader.purs#L22-L22)), kita dapat memanggilnya seperti ini:

```hs
type Env = {
  baseUrl :: String
}

port :: Reader Env Int
port = pure 4005
```

Type signature di atas mendeskripsikan bahwa kita telah membuat sebuah Reader yang menerima sebuah object `Env` dan menghasilkan `Int`. Lalu bagaimana cara menyuplai object `Env` dan mendapatkan nilai 4005? Dengan function `runReader`:

```hs
runReader :: Reader env a -> env -> a

env :: Env
env = { baseUrl: "http://localhost" }

result :: Int
result = runReader port env
```

`runReader` menerima Reader di argument pertamanya lalu disuplai dengan shared value `env` di argument kedua sehingga menghasilkan nilai `a`. Namun sampai contoh barusan kita belum juga menggunakan `Env` atau shared value yang kita suplai. Lalu bagaimana cara mendapatkannya? Dengan method `ask`!

```hs
insertPort :: Reader Env String
insertPort = do
  { baseUrl } <- ask
  pure $ baseUrl <> ":4005"

env :: Env
env = { baseUrl: "http://localhost" }

result :: Int
result = runReader insertPort env
-- "http://localhost:4005"
```

Contoh yang lebih real world:

```hs
fetchAuthedUser :: Reader Env User
fetchAuthedUser = do
  env  <- ask
  user <- Ajax.get (env.baseUrl <> "/my-details")
  pure user

fetchArticles :: Reader Env (Array Article)
fetchArticles = do
  env      <- ask
  user     <- fetchAuthedUser
  articles <- Ajax.get (env.baseUrl <> "/article?userId=" <> user.id)
  pure articles

fetchAllData :: Reader Env Data
fetchAllData = do
  articles <- fetchArticles
  tags     <- fetchTags
  ...
  ...
  pure someData


allData = runReader fetchAllData { baseUrl: "http://localhost:4005" }
```

Ketika `ask` dipanggil, ia akan mengembalikan environment yang nantinya disuplai. Jadi kalau ingin mendapatkan environment, **_ask_ for it** 😉.

Kurang lebih inilah yang dimaksud dengan Reader Monad beserta penggunaan dasarnya. Lihat bagaimana dalam pemanggilan function `fetchAuthedUser` kita tidak pernah benar-benar secara eksplisit melempar object `Env` ke function argument-nya. Begitu pula dalam pemanggilan `fetchArticles`, tidak ada explicit passing object `Env`. Semua "disembunyikan" lewat Reader.

## Update Global Value?

Lumrahnya penggunaan Reader Monad hanyalah untuk **dibaca** value-nya, sesuai dengan namanya, Reader. Namun bukan berarti Reader tidak bisa diperlakukan seperti global setter. Kita tetap bisa meng-update environment yang tersimpan di Reader. Dengan sedikit hack: `Ref`.

`Ref` adalah struktur data yang _mutable_ di Purescript. Segala operasi yang berkaitan dengannya harus dijalankan di bawah payung Effect Monad, karena memang nature-nya yang tidak pure.

Contoh kecil adalah ketika aplikasi baru diakses lewat browser dan user belum melakukan login. Sedangkan object user ini perlu diakses di banyak tempat. Kita tetap bisa menempatkan object user ini sebagai shared value di Reader.

```hs
type Env = {
  currentUser :: Ref (Maybe User),
  baseUrl :: String
}

main :: Effect Unit
main = do
  -- Jalankan dengan nilai awal kosong
  currentUser <- Ref.new Nothing

  let env :: Env = {
    currentUser,
    baseUrl: "https://..."
  }

  runApp env
```

Nanti di bagian aplikasi lain, ketika user telah ter-autentikasi, barulah `currentUser` dapat di-update dan dibaca oleh function lain

```hs {hl_lines=[10,19]}
authenticate :: ∀ m.
  MonadAsk Env m =>
  MonadEffect m  =>
  m (Maybe User)
authenticate = do
  env <- ask
  case fetchProfile env.baseUrl of
    Left _     -> pure Nothing
    Right user -> do
      liftEffect $ Ref.write (Just user) env.currentUser
      pure (Just user)

navbar :: ∀ m.
  MonadAsk Env m =>
  MonadEffect m  =>
  m HTML
navbar = do
  env    <- ask
  mbUser <- liftEffect $ Ref.read env.currentUser
  case mbUser of
    Nothing   -> H.button_ [H.text "Login"]
    Just user -> H.text ("Welcome, " <> user.name)
```

## Wrap up

Kesimpulannya: Reder Monad sangat cocok untuk meneruskan informasi/value secara implisit untuk menghasilkan sebuah komputasi. Setiap kali kita memiliki sebuah value yang dibutuhkan di banyak tempat namun ingin menghasilkan hasil komputasi yang berbeda-beda, maka Reader Monad bisa menjadi salah satu solusi.

Saya harap tulisan ini dapat membantu dalam memahami motivasi di balik Reader Monad beserta penggunaannya. Terima kasih.
