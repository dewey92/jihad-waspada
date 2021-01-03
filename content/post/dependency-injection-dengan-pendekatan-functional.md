---
title: "Dependency Injection Dengan Pendekatan Functional"
date: 2020-03-19T02:39:00+01:00
description: "Dependency Injection selalu menjadi bahan obrolan yang gak pernah habis dibahas. Kita akan mengeksplor bersama apa alternatif yang ditawarkan oleh Functional Programming terhadap permasalahan DI"
images: ["/uploads/knit-di.jpg"]
image:
  src: "/uploads/knit-di.jpg"
  caption: Image by <a href="https://pixabay.com/users/Foundry-923783/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=869221">Foundry Co</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=869221">Pixabay</a>
tags: ["purescript", "haskell", "types", "dependency-injection", "functionalprogramming"]
categories: ["programming", "purescript", "functional programming"]
draft: false
---

Istilah Dependency Injection (DI) biasa digunakan untuk merujuk pada teknik dimana suatu object menyuplai dependencies dari object lain. Tujuan utamanya adalah agar object tersebut tidak terikat pada satu implementasi tertentu saja, membuatnya menjadi lebih fleksibel terhadap perubahan. Istilah ini cukup dikenal dalam dunia Object-Oriented Programming.

Berikut contoh kecil DI:

```ts {hl_lines=[7,10,13,24]}
interface FileSystemService {
  exists: (path: Path) => boolean;
  create: (path: Path, content: string) => void;
}

class AppGenerator {
  constructor(private fsService: FileSystemService) {} // [1]

  initProject(projectName: Path) {
    if (this.fsService.exists(projectName)) { // [2]
      throw new Error('Cannot create project. A file/dir already exists!')
    }
    this.fsService.create(projectName, "Generated content") // [2]
  }
}

// Later when calling
import fs from 'fs'

const fsService: FileSystemService = {
  exists: fs.existsSync,
  create: fs.writeFileSync,
}
const app = new AppGenerator(fsService); // [3]
app.initProject('my-awesome-project')
```

- Kita definisikan dependency di constructor class [1] dan berikan constraint dengan sebuah interface agar implementasi object `fsService` dapat diganti-ganti sesuka hati selama comply dengan interface tersebut.
- Dependency dapat diakses dengan keyword `this` [2].
- Dependency diberikan ketika object `AppGenerator` diinstansiasi [3].

Dengan pemodelan dependency lewat class constructor ini, class `AppGenerator` menjadi lebih fleksibel dan tidak terikat pada satu implementasi saja. We program to an interface, not implementation. Detil implementasi dapat diganti semisal ketika testing:

```ts
const fsMock: FileSystemService = {
  exists: jest.fn().mockReturnValue(false),
  create: jest.fn(),
};

const app = new AppGenerator(fsMock)
app.initProject('my-awesome-project')

expect(fsMock.create).toHaveBeenCalledWith('my-awesome-project')
```

Pertanyaannya: bagaimana jika dilakukan dengan pendekatan functional? Apakah bisa dilakukan tanpa adanya class? Cukup dengan function?

## Explicit Dependencies

Kita dapat melihat pada contoh di atas bahwa method `initProject` bergantung pada object `fsService` yang kemudian diakses melalui magic keyword "this". Bicara dalam konteks function, kita tak lagi dapat mengandalkan keyword "this" dan harus mencari cara lain agar tetap bisa mengakses `fsService`.

Satu-satunya cara adalah dengan menyuplai `fsService` langsung melalui function argument. Kenyataannya pure function memang mengharuskan kita untuk memberi semua data yang diperlukan melalui argument. Sehingga DI tetap bisa dilakukan dengan cara ini.

Well, technically dependencies == function arguments.

```ts
function initProject(fsService, projectName) {
                     ^^^^^^^^^
  if (fsService.exists(projectName)) {
    throw new Error('Cannot create project. A file/dir already exists!')
  }
  fsService.create(projectName, "Generated content")
}
```

No more `this`! Dependency didefinisikan langsung di function argument tanpa mengabaikan _substitutability_ object `fsService`.

```ts
initProject(fsService, 'my-awesome-project')
// or
initProject(fsMock, 'my-awesome-project')
```

## Partial Application

Anggap saja requirement berubah di kemudian hari. User butuh fitur _logging_ [1] untuk mengetahui apakah program berhasil dijalankan atau tidak. Lalu ada tambahan _options_ [2] juga agar user dapat mengkostumasi project dengan lebih leluasa.

```ts
initProject(fsService, logService, projectName, options) {
                       ^^^^^^^^^^               ^^^^^^^
                           [1]                    [2]
  logService.log('Initiating project...')

  if (fsService.exists(projectName) && !options.overwrite) {
    logService.error('Cannot create project. A file/dir already exists!')
    return
  }

  fsService.create(projectName, "Generated content")
  logService.success(`Successfully created project ${projectName}`)
}
```

Fungsi `initProject` kini membutuhkan dua buah external service (`fsService` dan `logService`) dan dua buah argument "biasa" (`projectName` dan `options`). Pada dasarnya mereka semua sama-sama function argument, hanya concern-nya saja yang berbeda. Kita bisa melakukan sedikit refactor untuk membuat external service terlihat lebih eksplisit dan terpisah dari yang lainnya.

```ts
function makeInitProject(fsService, logService) {     // dependencies
  return function initProject(projectName, options) { // "normal" arguments
    // ...
  }
}

// later
const initProject = makeInitProject(fsService, consoleService)
initProject('my-awesome-project', { overwrite: true })
```

Style ini mirip dengan OOP dimana `makeInitProject` seolah berperan sebagai `constructor` dalam menerima dan menyimpan dependencies dengan bantuan closure. Namun esensinya pendekatan ini tidaklah berbeda dengan solusi sebelumnya. _It's just a matter of style_.

## Type Class

Cara lain agar DI dapat diimplementasikan dengan cara functional adalah melalui [type class]({{< ref "./kenalan-dulu-sama-type-class.md" >}}). Tidak semua bahasa punya fitur type class. Beberapa yang mendukung ada Haskell, Idris, dan Purescript. Type class sendiri adalah kumpulan-kumpulan method tanpa implementasi &mdash; seperti interface &mdash; yang memungkinkan tercapainya ad-hoc polymorphism.

So, how does it look like?

```hs
type Path = String
type Content = String

class Monad m <= FsService m where -- [1]
  exists :: Path -> m Boolean
  create :: Path -> Content -> m Unit

initProject :: ∀ m.
  FsService m => -- [2]
  String -> m Unit
initProject projectName = do
  doesExist <- exists projectName -- [3]
  if doesExist
  then pure unit
  else create projectName "Generated content" -- [3]
```

- Pertama, kita harus definisikan type class `FsService` [1] yang memiliki dua buah method: `exists` dan `create`. Berikan `m` constraint Monad agar nantinya bisa kita berikan instance `Effect` (atau `Aff` untuk komputasi asynchronous) dan menjalankan **real** side-effect (IO operation).
- Berikan constraint `FsService` [2] pada function `initProject` agar kita bisa memanggil kedua buah method `FsService` [3] di dalamnya.

Dibandingkan dengan pendekatan-pendekatan sebelumnya di atas, kita tidak melihat dependency `fsService` terdefinisikan di function argument, hanya constraint class `FsService` di type signature saja. Lalu bagaimana kita bisa meng-inject implementasi konkrit dari class `FsService` kalau fungsi `initProject` sendiri tidak menerimanya di argument?

<details style="margin-bottom: 30px; text-align: center;">
  <summary>HINT 💡:</summary>
  <p>Compiler yang menyuplainya saat compile time</p>
</details>

Yang harus kita lakukan saat ini hanyalah membuat implementasi (instance) dari type class `FsService` semisal:

```hs
import Prelude
import Effect (Effect)
import Effect.Aff (Aff)
import Effect.Class (liftEffect)
import Node.Buffer as Buffer
import Node.Encoding (Encoding(..))
import Node.FS.Aff as FS

instance fsServiceAff :: FsService Aff where -- [1]
  exists path = FS.exists path -- [2]
  create path content = do -- [2]
    FS.mkdir path
    buffer <- liftEffect $ Buffer.fromString content UTF8
    FS.writeFile (path <> "/README.md") buffer
```

Mari bahas step-by-step:
- Kita memberikan `Aff` &mdash; sebuah Monad yang merepresentasikan komputasi asynchronous &mdash; instance `FsService` [1] dan beri nama instance tersebut `fsServiceAff`. Ini berarti method `exists` dan `create` bisa kita panggil di dalam konteks Aff (we'll do it below 👇🏼).
- Implementation details [2]. Jika dipanggil, program akan berinteraksi dengan file system.

Tinggal satu langkah tersisa: tempatkan function `initProject` ke dalam konteks `Aff` agar compiler dapat meng-inject instance `fsServiceAff`. Kita bisa menggunakan fungsi `launchAff_`.

```hs
import Effect.Aff (launchAff_)

main :: Effect Unit
main = launchAff_ do
  initProject "my-awesome-project"
```

Langkah barusan sangat penting karena jika tidak, compiler tidak akan bisa melakukan DI. Seandainya kita tempatkan fungsi `initProject` misal ke dalam konteks `Effect` seperti `main = pure $ initProject "my-awesome-project"`, compiler akan gagal mencari instance `FsService Effect` karena kita memang belum membuatnya: "No type class instance was found for `FsService Effect`", keluh compiler.

Kembali ke konteks `Aff`. Jika kita intip output hasil compile-nya, kita akan melihat bagaimana compiler melakukan DI untuk kita secara otomatis:

```js {hl_lines=[10,17]}
var FsService = function (exists, create) {
  this.exists = exists;
  this.create = create;
};

var fsServiceAff = new FsService(function () {
  return ...;
})

var initProject = function (dictFsService) { // Argument untuk menerima service (dep)
  return function (projectName) {            // Argument "biasa"
    return ...;
  };
};

var main = Effect_Aff.launchAff_(
  initProject(fsServiceAff)("my-awesome-project") // 💥 Inject `fsServiceAff`
              ^^^^^^^^^^^^
);
```

Bisa dicoba di [link ini](https://try.purescript.org/?gist=43ab6916f4b22e6b7893c005c62126b7&js=true).

### Dependency Baru

Sejauh ini kita dapat menyimpulkan bahwa compiler melakukan dependency injection saat compile time hanya dengan mendeklarasikan constraint type class pada type signature fungsi `initProject`. Artinya ketika kita punya service tambahan e.g `logService` dan `promptService`, kita hanya perlu mengubah type signature dan voila, dependencies ter-inject.

```hs
-- Service baru
data LogType = Debug | Error | Success
class Monad m <= LogService m where
  log :: LogType -> String -> m Unit

class Monad m <= PromptService m where
  prompt :: String -> m String

-- Buat instance-nya
instance logServiceAff :: LogService Aff where
  log = ...
instance promptServiceAff :: PromptService Aff where
  prompt = ...

-- Tambah constraint di type signature-nya
initProject :: ∀ m.
  FsService m =>
  LogService m =>    -- 👈🏻
  PromptService m => -- 👈🏻
  String -> m Unit
initProject projectName = do
```

Hasil compile-nya:

```js
var initProject = function (dictFsService) {
  return function (dictLogService) {
    return function (dictPromptService) {
      return function (projectName) {
        return ...;
      };
    };
  };
};

var main = Effect_Aff.launchAff_(
  initProject(fsServiceAff)(logServiceAff)(promptServiceAff)("my-awesome-project")
              ^^^^^^^^^^^^  ^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^
);
```

Nice! 🎉 🎉 🎉

### Ganti Implementasi

Di awal artikel saya menyebutkan salah satu benefit DI adalah kemampuan mengganti detil implementasi service tanpa perlu mengubah code yang menggunakannya. Dalam hal ini type class juga bisa diinstansiasi oleh tipe data lain. Contohnya ketika testing, kita ingin membuat "spy" terhadap method `FsService` dengan memanfaatkan Writer monad.

```hs
-- Berikan instance `FsService` kepada `Writer`
instance fsServiceWriter :: FsService (Writer [String] a) where
  exists path   = path /= "my-awesome-project"
  create path _ = tell [path]

spec = describe "initProject" do
  it "creates a project" do
    -- Jalankan `initProject` di dalam konteks `Writer`,
    -- dengan menempatkannya di dalam fungsi `execWriter`
    let calls = execWriter (initProject "my-awesome-project") in
    calls `shouldEqual` ["my-awesome-project"]
  it "fails creating a project" do
    let calls = execWriter (initProject "invalid") in
    calls `shouldEqual` []
```

_No problems whatsoever_.

---

## Putting it All Together

Dengan pattern ini, kita tidak lagi melakukan Dependency Injection dengan menyuplainya secara eksplisit lewat function argument seperti solusi di awal artikel. **Kita mengalihkannya ke type signature**. Biar compiler yang menuntaskan pekerjaan Dependency Injection-nya. Dari segi estetika, dependencies dan function arguments juga terpisah jelas, sesuai goal awal kita.

Menurut saya, ini win-win solution untuk masalah DI 😉

_To sum up, this may be our final code_:

```hs
main :: Effect Unit
main = launchAff_ program

program :: ∀ m.
  FsService m =>
  LogService m =>
  PromptService m =>
  m Unit
program = do
  projectName <- prompt "What's your project name: "
  when (not $ null projectName) do
    initProject projectName
```

Saya harap artikel ini bermanfaat dalam menambah wawasan teman-teman sekalian. Terima kasih banyak. _Stay well_ 🙂
