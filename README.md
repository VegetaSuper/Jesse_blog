---
title: Nextjs + tailwindcss 实现 大神 dan 一模一样的博客
date: "2024-02-23"
spoiler: Nextjs + tailwindcss 实战
---

大神 [dan](https://overreacted.io/) 的博客相信大家都看过，博客质量那是不用多说，懂的都懂。 说到博客样式，我比较喜欢这种简约风。其中博客中还支持组件交互效果。 所以我决定用 Nextjs + tailwindcss 实现一模一样的博客学习下。

技术选型:

- Nextjs
- tailwindcss
- MDXRemote

部署:

- vercel

## 项目文件结构：

```
|-- dan-blog
    |-- app
    |   |-- favicon.ico
    |   |-- globals.css
    |   |-- layout.tsx
    |   |-- page.tsx          博客列表(首页)
    |   |-- [slug]            博客详情
    |       |-- layout.tsx
    |       |-- markdown.css
    |       |-- page.tsx
    |-- components
    |   |-- blogList
    |   |   |-- list.tsx      博客列表
    |   |-- header
    |   |   |-- header.tsx    顶部header
    |   |-- homeLink
    |       |-- homeLink.tsx  顶部导航
    |-- data
    |   |-- post.ts           处理博客数据(博客列表、博客详情)
    |-- fonts
    |   |-- fonts.ts          全局字体
    |-- lib
    |   |-- utils.ts
    |-- posts                 写博客的地方
        |-- 博客文件夹1
        |   |-- index.mdx
        |-- 博客文件夹2
        |   |-- index.mdx
        |-- 博客文件夹3
        |   |-- index.mdx
        |-- 博客文件夹4
        |   |-- index.mdx
        |-- 博客文件夹5
            |-- components.js
            |-- 你的组件.js
            |-- index.mdx
            |-- 你的组件.js
            |-- 你的组件.js


```

我并不打算直接引入database来存储markdown文件，这样成本太大，你必须要选择一种数据库，还要编写数据库增删改查代码。对于一个本项目来说，甚至是博客这种小项目来说，得不偿失。

我规划将博客文章的markdown文件放在项目中，通过读取文件的方式来渲染博客文章。这样做的好处是，你可以直接在项目中编写markdown文件，push到github，vercel就会自动部署，你的博客就更新了。

nextjs的服务端组件能够很好的支持这种需求。我们可以通过服务端组件来读取文件，然后渲染到页面上。

在编写代码之前，你需要了解下nextjs基本工作原理，app router工作原理，以及动态路由工作原理。这样你才能更好的理解下面的代码。

## 博客列表

layout文件代码(src/app/layout.tsx)

```js
import type { Metadata } from "next";
import "./globals.css";
import { serif } from "@/fonts/fonts";
import Header from "@/components/header/header";

export const metadata: Metadata = {
  title: "overreacted - A blog by Dan Abramov",
  description: "Generated by create next app",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en" className={serif.className}>
      <body className="mx-auto max-w-2xl bg-[--bg] px-5 py-12 text-[--text]">
        <Header />
        <main>{children}</main>
      </body>
    </html>
  );
}

```

字体你可以选择你喜欢的样式，这里保持跟dan一模一样的字体

字体文件代码(src/fonts/fonts.ts)

```js
import { Montserrat, Merriweather } from "next/font/google";

export const sans = Montserrat({
  subsets: ["latin"],
  display: "swap",
  weight: ["400", "700", "900"],
  style: ["normal"],
});

export const serif = Merriweather({
  subsets: ["latin"],
  display: "swap",
  weight: ["400", "700"],
  style: ["normal", "italic"],
});
```

page作为首页，要渲染博客列表。在这之前需要先拿到博客列表数据。这里我将所有的博客md文件放在项目中的posts文件夹中(src/app/posts)。每一篇博客创建一个文件夹，文件夹中包含一个index.md文件，index.md就是写博客的地方，如果你需要交互组件，在当前文件夹中编写的组件代码，最后通过公共的components文件统一导出所有组件。

<img
  src="https://github.com/sunshineLixun/dan-blog/assets/15700015/376dce38-75b2-4517-b011-471c625d07e4"
  alt="20240223173442.jpg"
  width="50%"
/>

page文件代码(src/app/page.tsx)

```js
import BlogList from "@/components/blogList/list";

export default function Home() {
  return <BlogList />;
}
```

这里我将component单独抽离到app路径外，你可以更好的管理你的组件。app文件下的文件都是页面文件。这样划分，职责功能更加清晰。

BlogList组件(src/components/blogList/list.tsx)

```js
export default async function BlogList() {
  return <div className="relative -top-[10px] flex flex-col gap-8">...</div>;
}
```

接下来就需要获取博客列表数据，新建data文件夹(src/app/data/post.ts)，这里集中处理读取博客内容，获取到博客列表信息，和博客详情。可以理解为数据库操作。

读取全部文章数据：

```js
const rootDirectory = path.join(process.cwd(), "src", "posts");

export const getAllPostsMeta = async () => {
  // 获取到src/posts/下所有文件
  const dirs = fs
    .readdirSync(rootDirectory, { withFileTypes: true })
    .filter((entry) => entry.isDirectory())
    .map((entry) => entry.name);

  // 解析文章数据，拿到标题、日期、简介
  let datas = await Promise.all(
    dirs.map(async (dir) => {
      const { meta, content } = await getPostBySlug(dir);
      return { meta, content };
    }),
  );

  // 文章日期排序，最新的在最前面
  datas.sort((a, b) => {
    return Date.parse(a.meta.date) < Date.parse(b.meta.date) ? 1 : -1;
  });
  return datas;
};
```

```js
export const getPostBySlug = async (dir: string) => {
  const filePath = path.join(rootDirectory, dir, "/index.mdx");

  const fileContent = fs.readFileSync(filePath, { encoding: "utf8" });

  // gray-matter库是一个解析markdown内容，可以拿到markdown文件的meta信息和content内容
  const { data } = matter(fileContent);

	// 如果文件名是中文，转成拼音
  const id = isChinese(dir)
    ? pinyin(dir, {
        toneType: "none",
        separator: "-",
      })
    : dir;

  return {
    meta: { ...data, slug: dir, id },
    content: fileContent,
  } as PostDetail;
};
```

补全BlogList组件逻辑

```js
export default async function BlogList() {
  const posts = await getAllPostsMeta();

  return (
    <div className="relative -top-[10px] flex flex-col gap-8">
      {posts.map((item) => {
        return (
          <Link
            className="block scale-100 py-4 hover:scale-[1.005] active:scale-100"
            key={item.meta.id}
            href={"/" + item.meta.slug + "/"}
          >
            <article>
              <PostTitle post={item} />
              <p className="text-[13px] text-gray-700 dark:text-gray-300">
                {new Date(item.meta.date).toLocaleDateString("cn", {
                  day: "2-digit",
                  month: "2-digit",
                  year: "numeric",
                })}
              </p>
              <p className="mt-1">{item.meta.spoiler}</p>
            </article>
          </Link>
        );
      })}
    </div>
  );
}
```

<img
  src="https://github.com/sunshineLixun/dan-blog/assets/15700015/7b28c296-5887-41ba-b954-ea934e651c4a"
  alt="20240223174210.jpg"
  width="50%"
/>

你会发现博客标题颜色随着文章顺序变化，实现这个功能，单独抽离PostTitle组件：这里的逻辑并不是唯一，你可以根据自己的需求来实现你自己喜欢的颜色。主要逻辑就是`--lightLink` `--darkLink` 这两个css变量，你可以根据不同的逻辑来设置这两个变量的值。其中`--lightLink` `--darkLink`分别是日间/暗黑模式的变量，你可以根据你的需求来设置。

```js
import Color from "colorjs.io";

function PostTitle({ post }: { post: PostDetail }) {
  let lightStart = new Color("lab(63 59.32 -1.47)");
  let lightEnd = new Color("lab(33 42.09 -43.19)");
  let lightRange = lightStart.range(lightEnd);
  let darkStart = new Color("lab(81 32.36 -7.02)");
  let darkEnd = new Color("lab(78 19.97 -36.75)");
  let darkRange = darkStart.range(darkEnd);
  let today = new Date();
  let timeSinceFirstPost = (
    today.valueOf() - new Date(2018, 10, 30).valueOf()
  ).valueOf();
  let timeSinceThisPost = (
    today.valueOf() - new Date(post.meta.date).valueOf()
  ).valueOf();
  let staleness = timeSinceThisPost / timeSinceFirstPost;

  return (
    <h2
      className={[
        sans.className,
        "text-[28px] font-black",
        "text-[--lightLink] dark:text-[--darkLink]",
      ].join(" ")}
      style={{
        // @ts-ignore
        "--lightLink": lightRange(staleness).toString(),
        "--darkLink": darkRange(staleness).toString(),
      }}
    >
      {post.meta.title}
    </h2>
  );
}

```

## 博客详情

首页博客列表实现完了，接下来就是博客详情页面，渲染mdx文件内容，首推nextjs官方的MDXRemote组件，最重要的就是它支持引入自己编写的组件，这样就可以实现博客中的组件交互效果。

博客详情是nextjs动态路由最好的应用场景，你可以通过动态路由来渲染不同的博客详情页面。创建动态路由文件(src/app/[slug]/layout.tsx)。博客详情作为新页面自然是必须layout包裹的，所以在这里引入layout组件。

```js
import HomeLink from "@/components/homeLink/homeLink";

export default function DetailLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <>
      {children}
      <footer className="mt-12">

        <HomeLink />
      </footer>
    </>
  );
}

```

HomeLink组件没有关键逻辑，不用关心。

博客详情页面代码(src/app/[slug]/page.tsx)

```js
import { getAllPostsMeta, getPost } from "@/data/post";
import { MDXRemote } from "next-mdx-remote/rsc";
// 美化代码，支持代码颜色主题
import rehypePrettyCode from "rehype-pretty-code";
// 支持数学公式
import remarkMath from "remark-math";
import rehypeKatex from "rehype-katex";
import { sans } from "@/fonts/fonts";
import "./markdown.css";
import { getPostWords, readingTime } from "@/lib/utils";

export async function generateStaticParams() {
  const metas = await getAllPostsMeta();
  return metas.map((post) => {
    return { slug: post.meta.slug };
  });
}

export default async function PostPage({
  params,
}: {
  params: { slug: string };
}) {
  // 获取文章详情
  const post = await getPost(params.slug);
  let postComponents = {};

 // 提取自己编写的组件
  try {
    postComponents = await import(
      "../../posts/" + params.slug + "/components.js"
    );
  } catch (e: any) {
    if (!e || e.code !== "MODULE_NOT_FOUND") {
      throw e;
    }
  }

  const words = getPostWords(post.content);
  const readTime = readingTime(words);

  return (
    <article>
      <h1
        className={[
          sans.className,
          "text-[40px] font-black leading-[44px] text-[--title]",
        ].join(" ")}
      >
        {post.meta.title}
      </h1>
      <p className="mb-6 mt-2 text-[13px] text-gray-700 dark:text-gray-300">
        {new Date(post.meta.date).toLocaleDateString("cn", {
          day: "2-digit",
          month: "2-digit",
          year: "numeric",
        })}
      </p>

      <p className="mt-2 text-[13px] text-gray-700 dark:text-gray-300">
        字数：{words}
      </p>
      <p className="mt-2 text-[13px] text-gray-700 dark:text-gray-300">
        预计阅读时间：{readTime}分钟
      </p>
      <div className="markdown mt-10">
        <MDXRemote
          source={post?.content || ""}
          // 关键：导入组件之后，博客文章才会有组件交互效果
          components={{
            ...postComponents,
          }}
          options={{
            parseFrontmatter: true,
            mdxOptions: {
              // @ts-ignore
              remarkPlugins: [remarkMath],
              rehypePlugins: [
                // @ts-ignore
                rehypeKatex,
                [
                  // @ts-ignore
                  rehypePrettyCode,
                  {
                    theme: "material-theme-palenight",
                  },
                ],
              ],
            },
          }}
        />
      </div>
    </article>
  );
}

```

在BlogList组件中，我们使用`Link`组件包裹所有内容，并设置`href`属性为`"/" + item.meta.slug + "/"`，这样就可以通过动态路由来渲染不同的博客详情页面。

```js
<Link
	href={"/" + item.meta.slug + "/"}
>
```

函数generateStaticParams 函数可以与动态路由段结合使用，在构建时静态生成路由，而不是在请求时按需生成路由。这样可以提高页面加载速度。

获取文章详情方法(src/data/posts.ts)

```js
export async function getPost(slug: string) {
  const posts = await getAllPostsMeta();
  if (!slug) throw new Error("not found");
  const post = posts.find((post) => post.meta.slug === slug);
  if (!post) {
    throw new Error("not found");
  }
  return post;
}
```

提取自己编写的组件，丢给MDXRomote组件。

```js
  try {
    postComponents = await import(
      "../../posts/" + params.slug + "/components.js"
    );
  } catch (e: any) {
    if (!e || e.code !== "MODULE_NOT_FOUND") {
      throw e;
    }
  }
```

文章markdown渲染美化css

```css
.markdown {
  line-height: 28px;
  --path: none;
  --radius-top: 12px;
  --radius-bottom: 12px;
  --padding-top: 1rem;
  --padding-bottom: 1rem;
}

.markdown p {
  @apply pb-8;
}

.markdown a {
  @apply border-b-[1px] border-[--link] text-[--link];
}

.markdown hr {
  @apply pt-8 opacity-60 dark:opacity-10;
}

.markdown h2 {
  @apply mt-2 pb-8 text-3xl font-bold;
}

.markdown h3 {
  @apply mt-2 pb-8 text-2xl font-bold;
}

.markdown h4 {
  @apply mt-2 pb-8 text-xl font-bold;
}

.markdown :not(pre) > code {
  border-radius: 10px;
  background: var(--inlineCode-bg);
  color: var(--inlineCode-text);
  padding: 0.15em 0.2em 0.05em;
  white-space: normal;
}

.markdown pre {
  @apply -mx-4 mb-8 overflow-y-auto p-4 text-sm;
  clip-path: var(--path);
  border-top-right-radius: var(--radius-top);
  border-top-left-radius: var(--radius-top);
  border-bottom-right-radius: var(--radius-bottom);
  border-bottom-left-radius: var(--radius-bottom);
  padding-top: var(--padding-top);
  padding-bottom: var(--padding-bottom);
}

.markdown pre code {
  width: auto;
}

.markdown blockquote {
  @apply relative -left-2 -ml-4 mb-8 pl-4;
  font-style: italic;
  border-left: 3px solid hsla(0, 0%, 0%, 0.9);
  border-left-color: inherit;
  opacity: 0.8;
}

.markdown blockquote p {
  margin: 0;
  padding: 0;
}

.markdown p img {
  margin-bottom: 0;
}

.markdown ul {
  margin-top: 0;
  padding-bottom: 0;
  padding-left: 0;
  padding-right: 0;
  padding-top: 0;
  margin-bottom: 1.75rem;
  list-style-position: outside;
  list-style-image: none;
  list-style: disc;
}

.markdown li {
  margin-bottom: calc(1.75rem / 2);
}

.markdown img {
  @apply mb-8;
  max-width: 100%;
}

.markdown pre [data-highlighted-line] {
  margin-left: -16px;
  padding-left: 12px;
  border-left: 4px solid #ffa7c4;
  background-color: #022a4b;
  display: block;
  padding-right: 1em;
}
```

至此逻辑已经全部完成，这个博客项目还比较简答。 其核心逻辑就是 读取markdown文件-MDXRemote组件渲染mdx文件内容。MDXRemote组件还支持丰富的插件功能，数学公式、图标展示。感兴趣的同学可以扩展。

完整代码在: [blog](https://github.com/sunshineLixun/dan-blog)
