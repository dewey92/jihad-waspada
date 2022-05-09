---
title: "Sembunyikan State-mu dengan State Monad"
date: 2020-03-28T10:04:00+01:00
description: "State monad sebagai pattern untuk meringankan state tracking dengan cara yang pure"
images: ["/uploads/desk-state-monad.jpg"]
image:
  src: "/uploads/desk-state-monad.jpg"
  caption: Image by <a href="https://pixabay.com/users/punttim-3413364/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1245954">Tim Gouw</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1245954">Pixabay</a>
tags: ["purescript", "haskell", "types", "monad", "functionalprogramming"]
categories: ["programming", "purescript", "functional programming"]
draft: false
---

Ceritanya lagi buat aplikasi bank kecil-kecilan.

```purs
withdraw :: Int -> Int
withdraw amount = amount
```

Argument function `withdraw` adalah jumlah yang ingin kita ambil dan nilai kembaliannya kita asumsikan sebagai uang yang keluar dari mesin ATM.

Namun function tersebut kurang _real world_. Untuk melakukan penarikan uang, mesin harus menghitung beberapa faktor yang telah ditentukan oleh pihak bank seperti jumlah penarikan, biaya penarikan, saldo, dll. Sedangkan tak ada pengecekan apapun di function tersebut. Mungkin harus kita buat versi yang lebih baik.

```purs
type Amount = Int
type Balance = Int

withdraw :: Amount -> Balance -> Tuple (Maybe Amount) Balance
withdraw amount balance =
  let minBalance = 50 in
  let afterBalance = balance - amount in

  if afterBalance >= minBalance
    then Just amount
    else Nothing
```

Sekarang function `withdraw` akan cek terlebih dahulu apakah sisa saldo setelah penarikan masih lebih dari 50. Jika demikian, penarikan berhasil. Namun sayangnya sisa saldo setelah penarikan tidak berubah sama sekali. Mungkin perlu diiterasi lagi supaya mengembalikan dua hal sekaligus: jumlah yang keluar dari ATM, dan hasil saldo akhir setelah transaksi.

```purs
type Amount = Int
type Balance = Int

withdraw :: Amount -> Balance -> Tuple (Maybe Amount) Balance
withdraw amount balance =
  let minBalance = 50 in
  let afterBalance = balance - amount in

  if afterBalance >= minBalance
    then Tuple (Just amount) afterBalance
    else Tuple Nothing balance
```

Dengan begini, nilai `balance` juga ikut dikomputasi setiap kali melakukan penarikan dan hasilnya dikembalikan ke caller.

```purs
transactions :: Int -> String
transactions balance =
  let (Tuple _        balance2) = withdraw 10 balance in
  let (Tuple mbAmount balance3) = withdraw 5 balance2 in

  case mbAmount of
    Just _  -> "Saldo terakhir Anda: " <> show balance3
    Nothing -> "Kismin lu!"

λ> transactions 100
"Saldo terakhir Anda: 85"

λ> transactions 10
"Kismin lu!"
```

Pattern yang langsung terlihat dari penggunaan fungsi `withdraw` adalah nilai `balance` yang dioper-oper secara eksplisit dari satu function ke function lainnya, yang nilainya berpotensi berubah di setiap pemanggilan `withdraw`. Nilai yang berubah-ubah ini bisa kita namakan dengan State.

Sehingga, dalam domain studi kasus "bank" ini, saldo atau `balance` adalah State yang harus kita maintain.

Balik ke permasalahan code, walaupun dalam banyak hal code mungkin akan lebih mudah dicerna dengan explicit passing, namun dalam hal ini akan sangat tidak readable bila harus terus melakukan _destructuring_ Tuple dan menyuplai State ke function berikutnya.

State monad bisa membantu menyembunyikannya.

## Definisi State Monad

```purs
type State s a = s -> Tuple a s
```

Sebetulnya function `withdraw` sudah menyerupai pattern State monad. Perhatikan type signature-nya dan fokus pada Balance, karena ia adalah state yang ingin kita maintain.

```purs
type Amount = Int
type Balance = Int

withdraw :: Amount -> Balance -> Tuple (Maybe Amount) Balance
                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

withdraw :: Amount -> State Balance (Maybe Amount)
```

Jadi sebenarnya State monad hanyalah sebuah function yang mengambil state dan mengembalikan state berikutnya, dibarengi dengan intermediate value (biasanya berupa hasil komputasi yang bergantung pada state).

```purs
state -> Tuple intermediateValue nextState
```

## Refactor

> Potongan code State yang muncul mulai section ini diambil dari module `Control.Monad.State`.

Mari refactor function `withdraw` menggunakan State monad.

```purs
type Amount = Int
type Balance = Int

withdraw :: Amount -> State Balance (Maybe Amount)
withdraw amount = do
  balance <- get

  let minBalance = 50
  let afterBalance = balance - amount

  if afterBalance >= minBalance
    then do
      put (balance - amount)
      pure $ Just amount
    else pure Nothing
```

Ada dua method State monad yang muncul di sini: `get` yang mengambil nilai state terbaru, dan `put` yang meng-overwrite nilai state.

Dengan style ini, function `transactions` menjadi lebih singkat dan kita tak perlu lagi passing state secara eksplisit.

```purs
type Amount = Int
type Balance = Int

deposit :: Amount -> State Balance Unit
deposit amount = modify (\s -> s + amount)
-- `modify` sama seperti `put`, namun menerima callback

transactions :: State Balance String
transactions = do
  deposit 8
  _      <- withdraw 10
  mbAmnt <- withdraw 5
  balance <- get
  pure $ case mbAmnt of
    Just _  -> "Saldo terakhir Anda: " <> show balance
    Nothing -> "Kismin lu!"
```

Dan untuk menjalankannya, kita hanya perlu memanggil `runState` (atau `evalState` atau `execState`, tergantung kebutuhan) dan menyuplainya dengan initial state.

```purs
initialBalance = 100

λ> runState transactions initialBalance
Tuple "Saldo terakhir Anda: 93" 93

λ> evalState transactions initialBalance
"Saldo terakhir Anda: 93"

λ> execState transactions initialBalance
93
```

Proses modifikasi dan passing state ini hanya terjadi di dalam operasi monad (ketika memanggil `runState`), menjaga program tetap pure dan bebas dari side effect. Mungkin ini salah satu perbedaan yang cukup mencolok dibandingkan dengan pendekatan OOP yang menggunakan class properties sebagai state variable. Atau pada pendekatan prosedural yang memanfaatkan mutable variable untuk menyimpan perubahan state.

---

_All in all_, State monad memberikan jalan alternatif yang pure untuk melakukan komputasi yang "stateful". Menambahkan state pada function `a -> b` cukup dengan mengubahnya menjadi `a -> State s b`, dan kita sudah mendapatkan function `get`, `put` dan `modify` secara gratis.

Semoga bermanfaat.
