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

Pada dasarnya DI dapat diilustrasikan dengan code berikut (_oversimplified and contrived_):

```ts
interface FileSystemService {
  exists: (path: Path) => boolean;
  create: (path: Path, content: string) => void;
  // ... etc
}

class CommandLineApp {
  constructor(fsService: FileSystemService) {
    this.fsService = fsService
  }

  initProject(projectName: Path) {
    if (this.fsService.exists(projectName)) {
      throw new Error('Cannot create project. A file/dir already exists!')
    }
    this.fsService.create(projectName)
  }
}

// later when calling
import fs from 'fs'

const fsService: FileSystemService = {
  exists: fs.existsSync,
  create: fs.writeFileSync,
  // ... etc
}
const app = new CommandLineApp(fsService);
app.initProject('my-awesome-project')
```

Dengan teknik ini, implementasi object `fsService` dapat diganti-ganti sesuka hati asal sesuai dengan interface yang telah ditentukan, semisal untuk kebutuhan testing.

```ts
const fsMock: FileSystemService = {
  exists: jest.fn().mockReturnValue(false),
  create: jest.fn(),
  // ... etc
};

const app = new CommandLineApp(fsMock)
app.initProject('my-awesome-project')

expect(fsMock.create).toHaveBeenCalledWith('my-awesome-project')
```

Dependency Injection terlihat begitu mudah sampai suatu saat kita mulai bertanya-tanya: bagaimana jika dilakukan dengan pendekatan functional yang notabene segalanya terdiri dari function? Apakah DI bisa tercapai tanpa adanya class?

## Explicit Dependencies

Pada contoh di atas kita dapat melihat bahwa method `initProject` bergantung pada object `fsService` agar bisa sepenuhnya bekerja. Object `fsService` dipanggil melalui magic keyword "this". Tanpa adanya class, kita kehilangan keyword tersebut sehingga `fsService` &mdash; yang "disimpan" melalui `constructor` &mdash; terkesan mustahil dipanggil.

Di lain sisi, kita tahu bahwa pure function hanya bergantung pada inputannya (function arguments) untuk melakukan suatu komputasi. Semua data atau object yang dibutuhkan oleh komputasi tersebut  harus disuplai sebagai input. Karenanya DI tetap bisa dilakukan dengan menyuplai dependencies secara eksplisit melalui function argument.

`initProject` membutuhkan `fsService`, maka `fsService` hanya perlu dinyatakan sebagai input function tersebut.

```ts
function initProject(fsService, projectName) {
  if (fsService.exists(projectName)) {
    throw new Error('Cannot create project. A file/dir already exists!')
  }
  fsService.create(projectName, "Yoo!")
}
```

Style-nya memang sedikit berbeda namun kemampuan untuk mensubstitusi object `fsService` masih tercapai.

```ts
initProject(fsService, 'my-awesome-project')
// or
initProject(fsMock, 'my-awesome-project')
```

## Partial Application

Yang mungkin menjadi tantangan tersendiri dari cara _explicit passing_ di atas adalah ketika suatu function memiliki beberapa dependency beserta input lainnya. Misal ketika kita ingin menambahkan fitur logging untuk menginformasikan progress program kepada user. Lalu ada tambahan options juga agar user dapat mengkostumasi pilihan mereka.

```ts
initProject(fsService, logService, projectName, options) {
  logService.log('Initiating project...')

  if (fsService.exists(projectName) && !options.overwrite) {
    logService.error('Cannot create project. A file/dir already exists!')
    return
  }

  fsService.create(projectName, "Yoo!")
  logService.success(`Successfully created project ${projectName}`)
}
```

Function `initProject` memiliki dua buah dependencies (`fsService` dan `logService`) dan dua buah argument biasa (`projectName` dan `options`). Mereka semua sama-sama merupakan function argument, hanya saja concern-nya yang berbeda. Perbedaan concern ini mungkin mendorong sebagian dari kita untuk melakukan refactor agar dependencies dapat dipisahkan dari argument lainnya. Dan salah satu cara memisahkan dua concern tersebut adalah dengan partial application.

```ts
function makeInitProject(fsService, logService) {
  return function (projectName, options) {
    // ...
  }
}

// later
const initProject = makeInitProject(fsService, console)
initProject('my-awesome-project', { overwrite: true })
```

Style ini juga kalau diperhatikan lebih mirip dengan gaya OOP, dimana `makeInitProject` seolah berperan sebagai `constructor` yang menerima dan menyimpan dependencies dengan bantuan closure. Namun pada akhirnya, pendekatan ini tidaklah berbeda dengan solusi sebelumnya. _It's just a matter of style_.

## Type Class

Cara lain agar DI dapat diimplementasikan dengan cara yang lebih functional adalah melalui teknik type class. Type class pada umumnya adalah kumpulan-kumpulan method tanpa implementasi &mdash; seperti interface &mdash; yang memungkinkan terciptanya polymorphism. Untuk penjelasan lebih detilnya, teman-teman bisa membaca artikel saya tentang [type class (dan apa perbedaannya dengan interface)]({{< ref "./kenalan-dulu-sama-type-class.md" >}}) dan bagaimana [cara kerjanya di balik layar]({{< ref "./type-class-dan-cara-kerjanya-di-balik-layar.md" >}}).

Lalu bagaimana type class dapat membantu dalam hal ini?

Saya rasa perlu sedikit demonstrasi dan penyegaran tentang type class terlebih dahulu. Saya juga memilih Purescript dalam penulisan contoh-contoh di bawah nanti karena dukungan fitur type class-nya.

Sebagai _warm-up_, code di bawah ini

```hs
class Show a where
  show :: a -> String

data Impl_one = Impl_one
data Impl_two = Impl_two

instance showImplOne :: Show Impl_one where
  show _ = "implementation one"

instance showImplTwo :: Show Impl_two where
  show _ = "implementation two"

shout :: ∀ a. Show a => a -> String
shout a = Data.String.toUpper (show a)

shoutImplOne = shout Impl_one
shoutImplTwo = shout Impl_two
```

ketika di-compile menghasilkan output:

```js {hl_lines=["22-26","28-29"],linenos=inline}
var Show = function (show) {
  this.show = show;
};
var show = function (dict) {
  return dict.show;
};

var Impl_one = (function () {
  ...
})();
var Impl_two = (function () {
  ...
})();

var showImplOne = new Show(function (v) {
  return "implementation one";
});
var showImplTwo = new Show(function (v) {
  return "implementation two";
});

var shout = function (dictShow) {
  return function (a) {
    return Data_String.toUpper(show(dictShow)(a));
  };
};

var shoutImplOne = shout(showImplOne)(Impl_one.value);
var shoutImplTwo = shout(showImplTwo)(Impl_two.value);
```

Perhatikan potongan code yang saya highlight. `shout` awalnya hanyalah sebuah _unary function_ (function dengan satu argument) namun menjadi _binary_ ketika di-compile, dengan parameter pertama berupa `dictShow`. `dictShow` akan diisi dengan implementation details oleh compiler ketika melakukan kompilasi (baris 28-29), dalam hal ini `showImplOne` dan `showImplTwo`. Implementation details tersebut dapat diubah-ubah sesuai konteks, persis seperti object `fsService` di atas tadi. Dengan kata lain, kita dapat menganggap `dictShow` sebagai dependency dari function `shout`.

Nah dengan analogi yang sama, kita seharusnya juga bisa merekonstruksi function `initProject` yang bergantung pada `fsService` menggunakan type class.

```hs
type Path = String

class FsService m where
  exists :: Path -> m Boolean
  create :: Path -> String -> m Unit

initProject :: ∀ m. FsService m => String -> m Unit
initProject projectName = -- ...
```

... yang akan menghasilkan output

```js {hl_lines=["12-16"],linenos=inline}
var FsService = function (exists, create) {
  this.exists = exists;
  this.create = create;
};
var exists = function (dict) {
  return dict.exists;
};
var create = function (dict) {
  return dict.create;
};

var initProject = function (dictFsService) {
  return function (projectName) {
      return ...;
  };
};
```

Cukup dengan memberikan constraint `FsService` kepada function `initProject`, kita berhasil membuatnya menerima dependency `dictFsService` yang kedepannya dapat diubah-ubah sesuai konteks.

Oh ya sebelum melanjutkan pembahasan lebih dalam, saya juga akan menambahkan constraint Monad terhadap type class tersebut supaya "do notation" dapat digunakan secara gratis.

```hs
type Path = String

class Monad m <= FsService m where
  exists :: Path -> m Boolean
  create :: Path -> String -> m Unit

initProject :: ∀ m. FsService m => String -> m Unit
initProject projectName = do
  doesExist <- exists projectName
  if doesExist
  then pure unit
  else create projectName "Yoo!"
```

Tinggal satu pekerjaan rumah tersisa: bagaimana memilih implementation details yang sesuai untuk function tersebut? Kita ingin bisa men-swap implementasi `dictFsService` dengan mudah untuk, anggap saja, kebutuhan testing.

## Layering App

> _Bear with me_. Section ini mungkin agak sedikit off dari pembahasan Dependency Injection. Namun saya merasa memahami Layering App secara fundamental sangat membantu dalam menangkap intuisi type class melakukan DI.

Matt Parsons pernah menulis artikel tentang [The Three Layer Haskell Cake](https://www.parsonsmatt.org/2018/03/22/three_layer_haskell_cake.html) dimana ia menjelaskan tentang ReaderT pattern dengan membagi aplikasi menjadi tiga bagian utama:

1. Layer 1, Imperative Shell: kita bisa saja membuat pure function sebanyak yang kita mau, tetapi pada akhirnya aplikasi harus tetap dijalankan dan menghasilkan effect (IO). Layer ini adalah layer terluar, yang berinteraksi dengan user input, request, configuration file, dan lain sebagainya. Testing di layer ini sangat tidak disarankan 😄
2. Layer 2, External Services dan Dependencies: layer ini berfungsi sebagai jembatan antara Layer 1 dan Layer 3. Di sini kita berfokus pada pembuatan "interface" atau type class service-service yang dibutuhkan aplikasi. Contohnya sudah ada pada section sebelumnya saat membuat `class FsService` 😉
3. Layer 3, Business Logic: layer ini harus sepenuhnya pure, tidak ada sangkut-pautnya dengan IO. Semua effectful function sudah terdefinisikan di Layer 2 dan dijalankan di Layer 1. Function `initProject` salah satu contohnya karena bersifat pure.

Baik, mari kita breakdown seperti apa sih code-nya nanti.

### Layer 1, Imperative Shell

Karena ini merupakan layer terluar yang berhubungan dengan effect di function main, kita harus bisa menerjemahkan "konteks" aplikasi kita yang pure ke dalam effect. "Konteks" ini bisa kita namakan `AppM`.

```hs
newtype AppM a = AppM (Effect a)

runAppM :: AppM ~> Effect
runAppM (AppM a) = a

-- instances boilerplate
derive newtype instance functorAppM :: Functor AppM
derive newtype instance applyAppM :: Apply AppM
derive newtype instance applicativeAppM :: Applicative AppM
derive newtype instance bindAppM :: Bind AppM
derive newtype instance monadAppM :: Monad AppM
derive newtype instance monadEffAppM :: MonadEffect AppM
```

`AppM` adalah konteks kita, dimana Layer 2 dan Layer 3 hidup di dalamnya.

```hs
main :: Effect Unit
main = runAppM do
  -- Layer 2 dan 3 akan dijalankan di dalam do block ini, seperti
  void $ initProject "my-awesome-project"
  -- Namun `initProject` adalah layer 3. Kita membutuhkan layer 2
  -- agar `initProject` dapat dijalankan dalam cangkang `AppM`!
  -- Layer 2 tidak terlihat di dalam do block ini, karena mostly
  -- ia berurusan dengan type.
```

### Layer 2, External Services dan Dependencies

Layer 2 hanyalah berupa kumpulan-kumpulan type class terhadap dependencies aplikasi. Kita sudah membuat `FsService` sebelumnya, dan saya akan sedikit mengubah nama class tersebut sekaligus memberikan instance `AppM` agar function-function yang ada di Layer 3 dapat "dimengerti" oleh Layer 1.

```hs
class Monad m <= ManageFileSystem m where
  exists :: Path -> m Boolean
  create :: Path -> String -> m Unit

instance manageFsAppM :: ManageFileSystem AppM where
  exists = ... -- implementation details, i.e using Node's fs.exists
  create = ... -- implementation details, i.e using Node's fs.writeFile
```

Karena ini hanyalah "interface" terhadap dunia luar, membuat mock implementasi untuk kebutuhan testing juga mudah.

```hs
-- Di Layer 1, definisikan "konteks"
newtype TestM a = TestM (Writer [String] a)

logTestM :: ∀ a. TestM a -> [String]
logTestM (TestM w) = execWriter w

-- Di Layer 2, mock it :))
instance manageFsTestM :: ManageFileSystem TestM where
  exists path   = path /= "my-awesome-project"
  create path _ = tell [path]

spec = describe "initProject" do
  it "creates a project" do
    calls <- pure $ logTestM (initProject "my-awesome-project")
    calls `shouldEqual` ["my-awesome-project"]
  it "fails creating a project" do
    calls <- pure $ logTestM (initProject "invalid")
    calls `shouldEqual` []
```

Di layer ini kita bebas membuat type class yang kita mau. Misal logger, atau user service, prompt, apapun itu yang mungkin berkaitan dengan effectful operations.

```hs
class Monad m <= ManageUser m where
  getUserById :: Int -> m (Maybe User)
  banUser :: UserId -> m Unit

data LogType = Debug | Error | Success
class Monad m <= Logger m where
  log :: LogType -> String -> m Unit

class Monad m <= Prompt m where
  prompt :: String -> m String
```

### Layer 3, Business Logic

Layer 3 ini sebenarnya layer yang paling asyik, karena semua function yang ada di sini adalah pure function, sama seperti function `initProject`. Layer ini abstrak, tidak terikat pada implementation details apapun. Sebagai hasilnya, layer ini mudah sekali untuk di-test (lihat contoh `TestM` di atas).

Yang saya sangat sukai dari pattern ini adalah bila ada sebuah function yang memiliki dependency lebih dari satu, maka kita hanya perlu menyatakannya lewat type signature sebagai constraint. Misal, function `initProject` kita dekorasi dengan logger dan prompt.

```hs {hl_lines=["2-4"]}
initProject :: ∀ m.
  ManageFileSystem m =>
  Logger m =>
  Prompt m =>
  String -> m Unit
initProject projectName = do
  log Debug "Initiating project..."
  doesExist <- exists projectName
  if doesExist
  then do
    answer <- prompt "Do you want to overwrite existing project? (y/n)"
    if answer /= "y"
    then log Error "Cannot create project. A file/dir already exists!"
    else createProject
  else createProject
  where
    createProject = do
      create projectName "Yoo!"
      log Success ("Successfully created project " <> projectName)
```

Dengan pattern ini, kita tidak lagi melakukan Dependency Injection dengan menyuplainya secara eksplisit lewat function argument seperti solusi di awal artikel tadi. **Kita mengalihkannya ke type signature**. Biar compiler yang menuntaskan pekerjaan Dependency Injection-nya. Dari segi estetika, dependencies dan function arguments juga terpisah jelas, sesuai goal awal kita.

Menurut saya, ini win-win solution untuk masalah DI 😉

## Putting it All Together

Memahami ketiga layer ini sangatlah penting untuk membuat aplikasi dengan pendekatan functional. Dependency Injection juga sudah di-manage oleh compiler _under the hood_. Fokus kita tertuju pada pembuatan type class terhadap dependencies (services) dan pure function saja. Selebihnya kita serahkan kepada compiler.

_To sum up, this may be our final code_:

```hs
main :: Effect Unit
main = runAppM program

program :: ∀ m.
  ManageFileSystem m =>
  Logger m =>
  Prompt m =>
  m Unit
program = do
  projectName <- prompt "What's your project name: "
  when (not $ null projectName) do
    initProject projectName

-- Untuk testing, tinggal ubah "context"-nya
test = runTestM program
```

Saya harap artikel ini bermanfaat dalam menambah wawasan teman-teman sekalian. Terima kasih banyak dan, _stay well_ 🙂