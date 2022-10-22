---
title: "Mengelola State Dengan Observer Pattern (Typescript)"
date: 2022-10-22T22:17:25+02:00
description: "Lewat pattern ini kita bisa mendapatkan update suatu nilai saat mengalami perubahan, gak perlu pake polling"
images: ["/uploads/listening.jpg"]
image:
  src: "/uploads/listening.jpg"
  caption: Image by <a href="https://pixabay.com/users/couleur-1195798/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2209152">Couleur</a> from <a href="https://pixabay.com//?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2209152">Pixabay</a>
tags: ["design pattern", "typescript"]
categories: ["programming"]
draft: false
---

Tujuan dari penulisan artikel ini adalah untuk mendemonstrasikan bahwa untuk me-manage state secara reaktif itu tidaklah sulit. Kita tak perlu harus selalu bergantung kepada third party library seperti Rxjs, Redux, Zustand, atau malah React. Saya percaya bahwa dengan memahami Observer Pattern saja sudah cukup untuk mengimplementasikan state yang reaktif.

## Studi Kasus

Oke langsung masuk ke studi kasus: anggap saja kita sedang membuat aplikasi video player, dimana selain user bisa memainkan video, ia juga bisa mengganti tema player. Jadi state kita ada dua: `isPlaying`, dan `theme`.

```ts
function playerUI() {
  type Theme = 'light' | 'dark'
  type State = {
    isPlaying: boolean
    theme: Theme
  }

  const state: State = {
    isPlaying: false,
    theme: 'light',
  }

  function togglePlay() {
    state.isPlaying = !state.isPlaying
  }

  function toggleTheme() {
    state.theme = state.theme === 'light' ? 'dark' : 'light'
  }

  return {
    state,
    togglePlay,
    toggleTheme,
  }
}
```

Looks good. Dari sisi consumer pun cukup jelas.

```ts
const ui = playerUI()

console.log(ui.state.isPlaying) // false
ui.togglePlay()
console.log(ui.state.isPlaying) // true
```

Wow emejing. Tapi kemudian requirement berubah: tim UX memutuskan untuk mengubah behavior dimana ketika player sedang dijalankan (`isPlaying == true`) elemen-elemen UI lain harus disembunyikan untuk memaksimalkan user experience.

```ts
function onPlayback() {
  ui.togglePlay()
  const { isPlaying } = ui.state

  if (isPlaying) {
    navbar.hide()
    sidebar.hide()
  } else {
    navbar.show()
    sidebar.show()
  }
}

<button onclick="onPlayback">▶⏸</button>
```

Kalau dilihat baik-baik, fungsi `onPlayback` kini mengatur semua side effect dari perubahan state `isPlaying`. Bagaimana kalau kita ingin menjalankan side effect lain seperti
- Mengirimkan event analytics saat player di-play
- Menampilkan iklan saat player di-pause (eeww)
- Meredupkan halaman saat di-play
- ... dan lain lain

Wah bakal terjadi banyak code coupling di dalam fungsi `onPlayback` nantinya karena ia tau terlalu banyak behavior dari elemen-elemen lain.

Kalau situasinya seperti ini, mungkin akan lebih baik jika semua side effect ini yang justru bereaksi terhadap perubahan state `isPlaying`. Jadi kita balik tanggungjawabnya. Kita ingin ada mekanisme dimana semua yang tertarik dengan nilai `isPlaying` bisa meninggalkan "nomor" mereka dan nomor-nomor ini akan ditelepon ketika terjadi perubahan terhadap nilai `isPlaying`. Seolah state ini reaktif. Mekanisme inilah yang disebut Observer Pattern.

Cara kerja Observer Pattern ini mirip seperti `addEventListener(event, callback)` dimana nomor (callback) yang kita tinggalkan akan ditelepon (dijalankan) saat suatu event terjadi. Mari kita ubah `playerUI`.

```ts {hl_lines=[6,"9-20",26],linenos=inline}
function playerUI() {
  // ...

  function togglePlay() {
    state.isPlaying = !state.isPlaying
    notifyListeners()
  }

  type Listener = (st: State) => void
  const listeners = new Set<Listener>()

  function notifyListeners() {
    listeners.forEach((cb) => cb(state))
  }

  function subscribe(cb: Listener) {
    listeners.add(cb)

    return { unsubscribe: () => listeners.delete(cb) }
  }

  return {
    state,
    togglePlay,
    toggleTheme,
    subscribe,
  }
}
```

Singkatnya, semua pihak yang tertarik dengan perubahan state harus menyediakan callback lewat fungsi `subscribe`. Callback-callback ini lalu didaftarkan ke dalam variable `listeners`. Dan ketika `isPlaying` berubah semuanya akan dijalankan. Jika tak terarik lagi, mereka bisa menjalankan fungsi `unsubscribe`.

Berikut contoh kode dari sisi consumer:

```ts
// ===========================================
// playerUI.js
// ===========================================
const ui = playerUI()
export default ui

// ===========================================
// Navbar.legacy.js
// ===========================================
ui.subscribe((state) => {
  if (state.isPlaying) {
    $('#navbar').hide()
  } else {
    $('#navbar').show()
  }
})

// ===========================================
// MainPage.react.tsx
// ===========================================
React.useEffect(function listenToPlayback() {
  const playerSubs = ui.subscribe((state) => {
    if (state.isPlaying) {
      dimPage(0.7)
    } else {
      showAds()
    }
  })

  return () => playerSubs.unsubscribe()
}, [])

// ===========================================
// analytics.redux.ts
// ===========================================
const middleware = (store) => {
  ui.subscribe((state) => {
    if (state.isPlaying) {
      store.dispatch({ ... })
    }
  })

  return (next) => (action) => {
    // ...
  }
}
```

## Refactor

Jika kita perhatikan fungsi `playerUI`, saya rasa fungsi ini menjalankan 2 hal yang berbeda sekaligus: mengelola listeners untuk reaktifitas, dan menyediakan behavior-nya sendiri (`togglePlay` dan `toggleTheme`). Harus kita pisahkan biar comply sama [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle).

Pertama, mari kita ekstrak kode pengelolaan state ke dalam fungsi sendiri, sebut saja `observable`.

```ts
function observable<T>(initValue: T) {
  type Listener = (st: T) => void
  const listeners = new Set<Listener>()

  let value = initValue

  function get() {
    return value
  }

  function set(fn: (currValue: T) => T) {
    value = fn(value)
    notifyListeners()
  }

  function notifyListeners() {
    listeners.forEach((cb) => cb(value))
  }

  function subscribe(cb: Listener) {
    listeners.add(cb)

    return { unsubscribe: () => listeners.delete(cb) }
  }

  return { get, set, subscribe }
}
```

Di sini kita hanya menambahkan function `get`. Tapi kenapa harus ada fungsi ini sedangkan bisa saja kita langsung mengembalikan `value` di posisi return? Sebenarnya bisa aja, asalkan `value` berbentuk object. Kalau primitive value malah jadi trikcy. Contoh:

```ts
function observable<T>(initValue: T) {
  let value = initValue
  // ...

  return { value, set, subscribe }
}

const obs = observable(0)
obs.set((val) => val + 1)
obs.value // 0 ❌
```

Kok bisa nilainya gak ter-update jadi 1? Ternyata begitu `set` dipanggil, hanya variable `value` di dalam function `observable` saja yang berubah, namun tidak di posisi return. Karena ketika fungsi `observable` dijalankan pertama kali, object yang di-return dievaluasi menjadi

```ts
return {
  value: 0,
  set,
  subscribe,
}
```

Oleh karena itu kita butuh fungsi `get` untuk mendapatkan `value` secara lazy (tidak langsung dievaluasi ketika return).

```ts
const obs = observable(0)
obs.set((val) => val + 1)
obs.get() // 1 ✅
```

Dan fungsi `playerUI` pun menjadi:

```ts
function playerUI() {
  const { get, set, subscribe } = observable<State>({
    isPlaying: false,
    theme: 'light',
  })

  function togglePlay() {
    set((state) => ({
      ...state,
      isPlaying: !state.isPlaying
    }))
  }

  function toggleTheme() {
    set((state) => ({
      ...state,
      theme: state.theme === 'light' ? 'dark' : 'light'
    }))
  }

  return {
    getState: get,
    subscribe,
    togglePlay,
    toggleTheme,
  }
}
```

Saya lampirkan code di atas ke dalam [link playground ini](https://www.typescriptlang.org/play?#code/GYVwdgxgLglg9mABHARgZwKYCcBuBDFAGwwB4AVAPgAoYwYoA1PQkDALkTIEpEBvAWABQiRFACeABwyIAMjDRQMYbIgC8iKgo7c1FRDjgwAJkJEQECxIXmLlWNGsTKA7ogDKGKCTkKl2alxCplae+sysjrT0TCwYQcKIoJCwCIgA5p5UPAIJIlieIFhI+LHBAL7xIknQ8EiYUFTAYBxUEIVYMazaPKp6OjkiIiUR6k1UwxiBuU5wsMBiPrbYaFnllYngNalgszDzi372WXzBIta+dmgAdMBwWACieBAAFlStKD16ECjj4ZNTIgqgmC1RSdRA6AgWBgKAw7w4Bzs2VOVhsh2ueCMRneUxR+SghSQvEQ4DQELQUJh7A0n1RF2WVyMGGIihxiCBgPW+MJfHSngANIh6oKyZDobD2UIgUJQbVEBJCHgxNgAKoASWOA1EkmkZGeGAAttJ1AByaxpZ5QE2IAA+iBNRjwWAA1ibguIpO4oHhFI4tSJ5AAFRViWhpDgoOBwYh4MAoqD6o3aRNxBLShLmMCWYkZKAigVC8mUiVlRyoTC4AjEEhub2Kaj+xBBkNhjjAZiYfnxlMcM0wC1Wrtp3EJWWpKBwNJpYjBpWalH1N4KH2TXQaRsiK5b5eKIeDQbNpWtxAAQh3GCuh9DYDSKLKXABkuBo82YNEk+nGD1hrhyOmi80OtV16dcUU3bcgL3fdRB7IUgKuBMfzUVRTXNS1rQAfntR0XWtXs0KtO8HzWZ88gKIoTmmXNaxXDhcygkRRQpcUMAY98pxnEM2InDivxTKCgXTTNLBAGBHAVJVVQ1KZhKgQt0EcUSriY4s4WE6ML0IScRyUnjP1nMRVkEXSP04ucphM3jvyNIyhCYq5SSLFjbOMmAENMvifyyIA) buat teman-teman yang mau ngulik lebih lanjut.

## Kesimpulan

Observer Pattern memungkinkan kita untuk mendapat "notifikasi" update suatu value saat terjadi perubahan, macam push notification. Makanya Observer Pattern ini cocok untuk masalah yang sifatnya one-to-many tanpa harus khawatir dengan code coupling. Si listener-nya pun bisa sebanyak mungkin. Seperti push notification, kita bisa "ubah settingannya" untuk tidak menerima update-an lagi dengan memanggil fungsi `unsubscribe`.

Sekian dulu, saya harap artikel ini bermanfaat. Cheers.
