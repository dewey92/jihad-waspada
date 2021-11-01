---
title: "Boolean: Bisa Jadi Bukan Teman Baikmu"
date: 2020-06-06T22:42:00+02:00
description: "Memodelkan behavior dengan boolean memang mudah. Namun apakah cukup sampai di situ?"
images: ["/uploads/truth.jpg"]
image:
  src: "/uploads/truth.jpg"
  caption: Image by <a href="https://pixabay.com/users/PDPics-44804/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=166853">PDPics</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=166853">Pixabay</a>
tags: ["typescript", "javascript", "react", "modelling"]
categories: ["programming", "modelling"]
draft: false
---

## Studi Kasus

Singkat cerita, Jum'at kemarin salah satu rekan kerja saya sedang membuat fitur "read more" untuk konten dengan tinggi lebih dari 300px (tidak ada tombol "read more" jika kurang). Ketika tombol tersebut di-click, seluruh isi konten baru akan ditampilkan.

```tsx
const ContentWrapper = () => {
  const [hasReadMoreBtn, setHasReadMoreBtn] = React.useState(false)
  const [isExpanded, setIsExpanded] = React.useState(false)
  const contentRef = React.useRef(null)

  React.useEffect(() => {
    if (contentRef.current.offsetHeight > 300) setHasReadMoreBtn(true)
  }, [])

  const expand = () => setIsExpanded(true)

  return (
    <div>
      <div ref={contentRef}>
        ...
      </div>
      {hasReadMoreBtn && <button onClick={expand}>read more</button>}
    </div>
  )
}
```

Cukup jelas. Namun ada masalah baru: `setHasReadMoreBtn(true)` dijalankan setelah render. Artinya, jika tinggi content ternyata melebihi 300px, _flash of content_ pun kemungkinan terjadi: awalnya user melihat seluruh isi content, lalu sepersekian detik kemudian tombol "read more" baru muncul

{{< figure src="/uploads/flash-of-content.gif" alt="flash of content" caption="flash of content" class="fig-center img-60" >}}

Ngakalinnya, jangan tampilkan konten ke user sebelum tinggi konten diketahui.

```tsx {hl_lines=[4,9,15]}
const ContentWrapper = () => {
  const [hasReadMoreBtn, setHasReadMoreBtn] = React.useState(false)
  const [isExpanded, setIsExpanded] = React.useState(false)
  const [isReady, setIsReady] = React.useState(false)
  const contentRef = React.useRef(null)

  React.useEffect(() => {
    if (contentRef.current.offsetHeight > 300) setHasReadMoreBtn(true)
    setIsReady(true)
  }, [])

  const expand = () => setIsExpanded(true)

  return (
    <div style={{ visibility: isReady ? 'visible' : 'hidden' }}>
      <div ref={contentRef}>
        ...
      </div>
      {hasReadMoreBtn && <button onClick={expand}>read more</button>}
    </div>
  )
}
```

Masalah selesai. Tapi entah kenapa ada sesuatu yang mengganjal. _It feels hacky_. **Masa iya harus butuh 3 buah state hanya untuk membuat fitur se-simple ini**. Belum lagi, dengan kombinasi tiga buah boolean saja, ada banyak kemungkinan yang bisa terjadi. Beberapa diantaranya justru tidak valid.

<div class="comparison-table">

| Kombinasi | Validitas & Penjelasan |
|:-:|:-:|
| `hasReadMoreBtn` ✅<br/>`isExpanded` ✅<br/>`isReady` ✅ | Valid. State ini terjadi ketika tinggi konten melebihi 300px, dan user sudah meng-klik tombol "read more" |
| `hasReadMoreBtn` ✅<br/>`isExpanded` ✅<br/>`isReady` ❌ | **Tidak valid**. Bagaimana mungkin `isExpanded` sudah bernilai true sedangkan `isReady` masih bernilai false |
| `hasReadMoreBtn` ✅<br/>`isExpanded` ❌<br/>`isReady` ✅ | Valid. State ini terjadi ketika konten dengan tinggi >300px sudah tersedia namun user belum meng-klik tombol "read more" |
| `hasReadMoreBtn` ✅<br/>`isExpanded` ❌<br/>`isReady` ❌ | Valid. State ini terjadi ketika konten dengan tinggi >300px belum ditampilkan ke layar |
| `hasReadMoreBtn` ❌<br/>`isExpanded` ✅<br/>`isReady` ✅ | **Tidak valid**. `isExpanded = true` (tombol "read more" ketika sudah di-click) hanya mungkin terjadi bila `hasReadMoreBtn` juga bernilai true |
| `hasReadMoreBtn` ❌<br/>`isExpanded` ✅<br/>`isReady` ❌ | **Tidak valid**. Sama seperti di atas |
| `hasReadMoreBtn` ❌<br/>`isExpanded` ❌<br/>`isReady` ✅ | Valid. konten sudah disajikan dan tingginya tidak melebihi 300px |
| `hasReadMoreBtn` ❌<br/>`isExpanded` ❌<br/>`isReady` ❌ | Valid. konten belum disajikan dan tingginya tidak melebihi 300px |

</div>

Pasti ada cara lain yang jauh lebih simple.

## Solusi

Setelah me-review ulang behavior fitur ini dengan secarik kertas untuk dicorat-coret, saya mendapati bahwa sebenarnya requirement-nya cukup simple:
  1. **Sembunyikan konten** sampai tinggi konten diketahui (hidden)
  2. Jika tinggi konten kurang dari 300px, **tampilkan seluruh isi konten** (expanded)
  3. Jika lebih,
      - **konten bisa di-expand** dengan tombol "read more" (expandable)
      - Setelah tombol "read more" di-klik, **tampilkan seluruh isi konten** (expanded)

Dari list ini terlihat bahwa secara behavior, konten hanya bisa memiliki salah satu dari ketiga state berikut: Hidden, Expandable, atau Expanded. Menyadari hal ini, spontan saja saya nyaut ke rekan saya: "BRO, PAKE ENUM/UNION!"

```tsx {hl_lines=[1,4]}
type ContentDisplay = 'hidden' | 'expandable' |'expanded'

const ContentWrapper = () => {
  const [contentDisplay, setContentDisplay] = React.useState<ContentDisplay>('hidden')
  const contentRef = React.useRef(null)

  React.useEffect(() => {
    setContentDisplay(contentRef.current.offsetHeight > 300
      ? 'expandable'
      : 'expanded'
    )
  }, [])

  const expand = () => setContentDisplay('expanded')
  const hasReadMoreBtn = contentDisplay === 'expandable'

  return (
    <div style={{ visibility: contentDisplay === 'hidden'
      ? 'hidden'
      : 'visible'
    }}>
      <div ref={contentRef}>
        ...
      </div>
      {hasReadMoreBtn && <button onClick={expand}>read more</button>}
    </div>
  )
}
```

Dengan pendekatan ini, kita telah mengeliminasi kemungkinan-kemungkinan state yang tidak valid sekaligus meningkatkan code readability. Kita bisa belajar dari kasus ini bahwa memang mudah memodelkan suatu behavior dengan boolean, namun saya rasa masih ada cara lain yang lebih tepat. Bila dalam memecahkan masalah **ada dua atau lebih boolean yang terlibat yang saling berkaitan**, mundurlah selangkah dan mulai pahami kembali requirement dari fitur yang ingin diimplementasi. Ambillah secarik kertas dan tulis semua kondisi yang mungkin muncul. Mungkin dengan enum atau union biasa solusimu jadi lebih simple 😉

Oh ya, saya sampai lupa ternyata saya juga pernah mengulas kasus serupa di artikel yang lain: [Tentang Loading State...](https://medium.com/codewey/tentang-loading-state-886dbb1cfe8d) 🙃

## Kesimpulan

Boolean bukanlah segalanya. Bahkan mungkin eksistensinya bisa tergantikan dengan enum atau untagged union.

```ts
enum Bool { true, false }

type Bool = 'true' | 'false'
```

Siapa yang tahu? Setidaknya, sudah [ada yang memodelkannya dengan "enum"](https://hackage.haskell.org/package/ghc-prim-0.6.1/docs/src/GHC.Types.html#Bool).

Sebagai penutup, video ini wajib ditonton bagi kamu yang ingin lebih mendalami tentang pemodelan business requirement ke dalam code:

{{< youtube PLFl95c-IiU >}}

<br/>
Stay well, my friend!
<br/>
<br/>

<style>
.comparison-table {
  font-size: 80%;
  overflow-x: auto;
}
.comparison-table code {
  font-size: 80% !important;
}
.comparison-table td:first-child {
  text-align: right !important;
  white-space: nowrap;
}
.comparison-table td:last-child {
  text-align: left !important;
}
</style>
