---
title: "Covariance and Contravariance in Typescript"
date: 2021-07-02T02:13:19+02:00
description: "How we can convert a union type into an intersection type using contravariance"
images: ["/uploads/intersection.jpg"]
image:
  src: "/uploads/intersection.jpg"
  caption: Image by <a href="https://pixabay.com/photos/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=984045">Free-Photos</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=984045">Pixabay</a>
tags: ["types", "typescript"]
categories: ["typescript", "programming", "type system"]
draft: false
---

Keep in mind that we're going to use the following principle throughout the article: **we can always substitute a type with its subtype**.

## Covariance

Suppose we have these simple types:

```ts
interface Blog {
  title: string
}

interface BlogWithImage extends Blog {
  img: string
}
```

We know that `BlogWithImage` is a subtype of `Blog`. So when there's a function that expects a `Blog` object, passing a `BlogWithImage` should not make the compiler bark.

```ts
function printTitle(blog: Blog) {
  console.log(blog.title)
}


printTitle({ title: 'A' } as Blog) // ✅
printTitle({ title: 'B', img: 'someURL' } as BlogWithImage) // ✅
```

And the following example should not typecheck:

```ts
function printImg(blog: BlogWithImage) {
  console.log(blog.img)
}


printImg({ title: 'A' } as Blog) // ❌
printImg({ title: 'B', img: 'someURL' } as BlogWithImage) // ✅
```

So from these instances, we can draw a conclusion that `BlogWithImage` is indeed a subtype of `Blog` as we can use it as a `Blog` substitute.

> `BlogWithImage` <: `Blog`

## Contravariance

However, when it comes to functions, things are a bit different. Suppose we have `fetchBlog` function which takes a callback:

```ts
await function fetchBlog(callback: (b: Blog) => void) {
  const blog: Blog = await axios.get('...')

  callback(blog)
}

function printTitle (b: Blog): void {
  console.log(b.title)
}

function printTitleImg (b: BlogWithImage): void {
  console.log(b.title)
  console.log(b.img)
}

fetchBlog(printTitle) // ✅
fetchBlog(printTitleImg) // ❌
```

Providing `printTitleImg` will not typecheck and will throw at runtime cause the `img` field will not be present upon fetching the blog. The fact that we can't substitute `(b: Blog) => void` with `(b: BlogWithImage) => void` means that `(b: BlogWithImage) => void` is NOT a subtype of `(b: Blog) => void`

What about the other way around?

```ts
await function fetchBlogWithImage(callback: (b: BlogWithImage) => void) {
  const blogWI: BlogWithImage = await axios.get('...')

  callback(blogWI)
}

function printTitle (b: Blog): void {
  console.log(b.title)
}

function printTitleImg (b: BlogWithImage): void {
  console.log(b.title)
  console.log(b.img)
}

fetchBlogWithImage(printTitle) // ✅
fetchBlogWithImage(printTitleImg) // ✅
```

Passing `printTitle` typechecks and we won't encounter any runtime error since it only consumes `title` field from the `BlogWithImage` object and just doesn't care about the rest. So in this case `(blog: BlogWithImage) => void` is substitutable with `(blog: Blog) => void`, thus we can say that `(blog: Blog) => void` IS a subtype of `(blog: BlogWithImage) => void`.

> `(blog: Blog) => void` <: `(blog: BlogWithImage) => void`

**When it comes to function argument, the more general type will always be the subtype of the more specific one**. This is the opposite (contra!) of what we usually encounter 🙂

## Inferring in Contravariant Position

Suppose we have:

```ts
type BlogFn =
  | ((b: { title: string }) => void)
  | ((b: { img: string }) => void)

type WhatAmI = BlogFn extends (b: infer T) => void ? T : never
```

What will be the resulting type of `WhatAmI`?

We know that if there's an expression of `A extends B`, it's `A` that is the subtype and `B` that is the base type. So in this case, `BlogFn` is the subtype, and `(b: infer T) => void` is the base type.

And we know that:
- `{ title: string }` is more general than `{ title: string, img: string }`
- `{ img: string }` is more general than `{ title: string, img: string }`

Remember, when it comes to function argument, the more general type will always be the subtype of the more specific one. Thus it follows that:

- `(b: { title: string }) => void` is the subtype of (extends) `(b: { title: string, img: string }) => void`
- `(b: { img: string }) => void` is the subtype of (extends) `(b: { title: string, img: string }) => void`

From these premises, it makes total sense if `infer T` should give you `{ title: string, img: string }` as it is the very least compatible type for each of the `BlogFn` constituents. And indeed, the resulting type of `WhatAmI` is `{ title: string, img: string }` 🙂.

---

If you look closely, it seems like the compiler converts a union type into an intersection type when inferring in contravariant position.

```ts
type BlogFn =
  | ((b: { title: string }) => void)
  | ((b: { img: string }) => void)

type IntersectionType = BlogFn extends (b: infer T) => void
  ? (b: T) => void
  : never
/**
 * `IntersectionType` results in
 *
 * (b: { title: string, img: string }) => void
 */
```

Interesting! Why don't we just create a utility type that converts union types to intersection types?! 💡

```ts
type U2I<U> =
  (U extends unknown ? (k: U) => void : never) extends ((k: infer I) => void) ? I : never
```

The `U extends unknown` expression makes sure that we distribute/iterate over the union `U`. As stated in the [documentation](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#distributive-conditional-types), "When conditional types act on a generic type, they become distributive when given a union type." And the rest is similar to what we have just discussed.

```ts
type TitleAndImg = U2I<{ title: strnig } | { img: string }>
// { title: string, img: string }
```

I hope you find this article useful. Stay well!
