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

```hs
withdraw :: Int -> Int
withdraw amount = amount
```

Nilai kembalian function `withdraw` kita asumsikan sebagai uang yang keluar dari mesin ATM.

Namun function tersebut sebenernya _useless_. Pertama, karena gak ngapa-ngapain selain mengembalikan uang sejumlah yang dimasukkan. Kedua, karena kurang _real world_. Untuk melakukan penarikan uang, mesin harus menghitung beberapa faktor yang telah ditentukan oleh pihak bank seperti jumlah penarikan, biaya penarikan, saldo, dll. Sedangkan tak ada pengecekan apapun di function tersebut. Mungkin harus kita buat versi yang lebih baik.

```hs
withdraw :: Int -> Int -> Maybe Int
withdraw amount balance =
  if balance > 50 && amount < balance
    then Just amount
    else Nothing
```

Misal saldo saya ada 100 dan saya mengambil uang sebesar 10, maka uang yang keluar dari mesin adalah 10. _Works_, karena saldo lebih dari 50. Bagaimana dengan balance saya setelah penarikan? Oh pasti jadi 90 dong. Ermm, tapi apa ada bukti? Function tersebut gak mengubah nilai saldo sedikitpun. Ia hanya mengembalikan "uang" yang kita inputkan.

Oke, mungkin perlu diiterasi lagi supaya mengembalikan dua hal sekaligus: jumlah yang keluar dari ATM, dan hasil saldo akhir setelah transaksi.

```hs
withdraw :: Int -> Int -> Tuple (Maybe Int) Int
withdraw amount balance =
  if balance > 50 && amount < balance
    then Tuple (Just amount) (balance - amount)
    else Tuple Nothing balance
```

Dengan begini, nilai `balance` juga ikut dikomputasi setiap kali melakukan penarikan dan hasilnya dikembalikan ke caller.

```hs
transactions :: Int -> String
transactions balance =
  let (Tuple _    balance2) = withdraw 10 balance in
  let (Tuple amnt balance3) = withdraw 5 balance2 in
  case amnt of
    Just _  -> "Saldo terakhir Anda: " <> show balance3
    Nothing -> "Kismin lu!"

λ> transactions 100
"Saldo terakhir Anda: 85"

λ> transactions 10
"Kismin lu!"
```

Pattern yang langsung terlihat dari penggunaan fungsi `withdraw` adalah nilai `balance` yang dioper-oper secara eksplisit dari satu function ke function lainnya, yang nilainya berpotensi berubah di setiap pemanggilan `withdraw`. Nilai yang berubah-ubah ini bisa kita namakan dengan State.

Sehingga, dalam domain studi kasus "bank" ini, saldo atau `balance` adalah State yang harus kita maintain.

Balik ke permasalahan code, walaupun dalam banyak hal code mungkin akan lebih mudah dicerna dengan explicit passing, namun tentunya dalam hal ini akan sangat tidak readable bila harus terus melakukan _destructuring_ Tuple dan menyuplai State ke function berikutnya.

State monad bisa membantu menyembunyikannya.

## Definisi State Monad

Sebetulnya function `withdraw` sudah menyerupai pattern State monad. Perhatikan type signature-nya dan fokus pada Balance, karena ia adalah state kita.

```hs
type Amount = Int
type Balance = Int

withdraw :: Amount -> Balance -> Tuple (Maybe Amount) Balance
                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Lalu kita extract type signature yang digarisbawahi:

```hs
type BalanceState = Balance -> Tuple (Maybe Amount) Balance

withdraw :: Amount -> BalanceState
```

dan bila `BalanceState` digeneralisir, akan menjadi:

```hs
type State s a = s -> Tuple a s

withdraw :: Amount -> State Balance (Maybe Amount)
```

Dan jadilah definisi State monad: sebuah function yang mengambil state dan mengembalikan state berikutnya, dibarengi dengan intermediate value (biasanya berupa hasil komputasi yang bergantung pada state).

```hs
state -> Tuple intermediateValue nextState
```

## Refactor

> Potongan code State yang muncul mulai section ini diambil dari module `Control.Monad.State`.

Mari refactor function `withdraw` menggunakan State monad.

```hs
withdraw :: Int -> State Int (Maybe Int)
withdraw amount = do
  balance <- get
  if balance > 50 && amount < balance
    then do
      put (balance - amount)
      -- atau `modify (\st -> st - amount)`
      pure $ Just amount
    else pure Nothing
```

Ada beberapa function State monad di sini. Pertama `get`, yang mengembalikan nilai state terbaru. Kedua ada function `put`, yang berguna untuk meng-overwrite nilai state. Ketiga ada `modify`, yang mirip dengan `put` namun alih-alih menerima function.

Dengan style ini, function `transactions` menjadi lebih singkat dan kita tak perlu lagi passing state secara eksplisit.

```hs
deposit :: Int -> State Int Unit
deposit amount = modify_ (_ + amount)

transactions :: State Int String
transactions = do
  deposit 8
  _    <- withdraw 10
  amnt <- withdraw 5
  balance <- get
  pure $ case amnt of
    Just _  -> "Saldo terakhir Anda: " <> show balance
    Nothing -> "Kismin lu!"
```

Dan untuk menjalankannya, kita hanya perlu memanggil `runState` (atau `evalState` atau `execState`, tergantung kebutuhan) dan menyuplainya dengan initial state.

```hs
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

_All in all_, State monad memberikan jalan alternatif yang pure untuk melakukan komputasi yang "stateful". Menambahkan state pada function `a -> b` cukup dengan mengubahnya ke `a -> State s b`, dan kita sudah mendapatkan function `get`, `put` dan `modify` secara gratis.

Semoga bermanfaat.