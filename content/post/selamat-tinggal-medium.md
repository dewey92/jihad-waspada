---
title: "Selamat Tinggal Medium"
date: 2019-06-30T10:57:25+02:00
description: Medium bagus sih, cuman kayaknya lebih asik bikin blog sendiri 🥳
images: ["/uploads/writing.jpg"]
image:
  src: "/uploads/writing.jpg"
  caption: <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px" href="https://unsplash.com/@nickmorrison?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Nick Morrison"><span style="display:inline-block;padding:2px 3px"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-2px;fill:white" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M10 9V0h12v9H10zm12 5h10v18H0V14h10v9h12v-9z"></path></svg></span><span style="display:inline-block;padding:2px 3px">Nick Morrison</span></a>
tags: ["random"]
categories: ["random"]
draft: false
---

Setelah mempertimbangkan matang-matang akhirnya saya memutuskan untuk lepas menulis dari Medium. Jangan salah sangka, saya sebenarnya suka banget nulis di Medium, bahkan saya [sudah menulis di Medium sejak tahun 2017](https://medium.com/@Dewey92) 😃

## Plus Minus Medium

Dari sudut pandang saya pribadi, banyak hal yang membuat platform ini cocok untuk menulis:

- Editor yang nyaris sempurna. Gak sedikit library editor WYSIWYG mengambil inspirasi dari Medium lho!
- Publikasi. Kita bisa membuat publikasi kita sendiri maupun berkontribusi di publikasi orang lain. Apa keuntungan menulis di publikasi orang lain? Jika kita menulis artikel di publikasi yang sudah dikenal komunitas, kemungkinan dibacanya akan sangat tinggi dan impact bagusnya adalah reputasi kita akan semakin naik.
- Exposure. Artikel-artikel yang kita tulis bisa direkomendasikan secara otomatis oleh sistem ke pembaca
- Integrasi ke gist. Tinggal copy link gist, muncul.
- Statistik. Gak perlu setup analytics lain-lain, _chart_ dan _insight_ dari Medium sendiri sudah cukup bagus.

Tapi saya juga punya beberapa hal yang membuat saya mau _move away_ dari Medium:

- Sebagian besar isi blog saya membahas konten technical. Dalam hal ini, editor Medium belum cukup bagus untuk menampilkan code yang saya tulis. Gak ada built-in _syntax highlighting_, block `<code> ... </code>` nya juga seringkali terlalu besar sehingga membaca code lewat mobile akan sangat sulit.

    {{< figure src="/uploads/medium-mobile.png" alt="code di Medium sulit dibaca" caption="Sulit membaca di mobile" class="fig-center img-60" >}}

    Saya **harus** menggunakan gist dan meng-import-nya _only to show code colours and enhance readability_ 🤦🏻‍♀️
- _Paywalls_. Sah-sah aja sih, _just doesn't work for me_.
- _Personal Branding_. Beberapa teman saya di Facebook memberikan saran ke saya lebih baik membuat blog pribadi jika memang suka menulis. Punya blog sendiri itu bisa meningkatkan _personal branding_ saya sebagai developer. _My content belongs to me_.
- _Reusable_. Saat ini saya menggunakan Hugo dan seluruh artikel di sini berformat Markdown yang artinya jika suatu saat nanti saya pindah platform (misal dari Hugo ke Gatsby) saya tidak harus menulis semua artikel yang pernah ditulis.

Awalnya berasa mager banget pindah dari Medium dan harus setup blog sendiri. Tapi pas dijalani ternyata gak sulit. Tinggal beli domain sendiri (~10 menit), install Hugo dan theme-nya (~30 menit), tarok di github (&lt; 1 menit), integrasi Netlify ke Github dan Domain (~5 menit), jadi deh blog baru 🎉

Untuk statistiknya, tinggal taro ID Google Analytics kita di `config.toml`. Untuk _commenting system_-nya, bisa install Disqus (tapi saya belum setup), atau bisa didiskusikan lewat Twitter dan Github. Karena pada dasarnya blog ini [open-source, semua bisa bikin kontribusi di Repo Github-nya](https://github.com/dewey92/jihad-waspada)

## Kesimpulan
Medium masih jadi platform yang sangat bagus untuk menulis. Tapi untuk menulis konten teknikal, masih banyak platform lain yang menyediakan fitur lebih bagus lagi seperti [Dev.to](https://dev.to), [Hashnode](https://hashnode.com/devblog), atau bahkan setup blog pribadi 🙂