---
draft: false

title: 使用Markdown编写漂亮的文档
subtitle: ""
description: ""

date: 2022-03-08T10:54:53+08:00
lastmod: 2022-03-08T10:54:53+08:00

author: "windvalley"
authorLink: ""

tags: ["markdown", "tips"]
categories: ["Hack"]

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: "MIT"
---

## 字符串高亮

```text
I need to highlight these ==very important words==.

或

I need to highlight these <mark>very important words</mark>.
```

I need to highlight these <mark>very important words</mark>.

## 粗体字和斜体字

```text
**bold text**
*italicized text*
This text is ***really important***.
```

**bold text**

_italicized text_

This text is **_really important_**.

## 删除线

```text
~~The world is flat.~~
```

~~The world is flat.~~

## 下划线

```text
Some of these words <ins>will be underlined</ins>.
```

Some of these words <ins>will be underlined</ins>.

## 水平线

使用 3 个或更多的`***`, `---` 或者`___` 打印水平线.

```text
Try to put a blank line before...

---

...and after a horizontal rule.
```

Try to put a blank line before...

---

...and after a horizontal rule.

## 脚注

```text
Here's a sentence with a footnote. [^1]

[^1]: This is the footnote.
```

Here's a sentence with a footnote. [^1]

[^1]: This is the footnote.

## Emoji

Gone camping! \:tent\: Be back soon.

That is so funny! \:joy\:

渲染后:

Gone camping! :tent: Be back soon.

That is so funny! :joy:

> 在这里找到更多 emoji 编码: https://gist.github.com/rxaviers/7360908

## 代码块

### 缩进代码块

一般使用 `4个空格缩进` 来展示代码块.

```text
Here's some regular text. And now a code block:

    print("Hello, world!")
    if True:
        print('true!')
```

渲染后:

Here's some regular text. And now a code block:

    print("Hello, world!")
    if True:
        print('true!')

> NOTE: 通过缩进 4 个空格来展示代码块, 无法高亮代码, 也不便于复制粘贴, 一般不推荐使用.

### 受防护的代码块

#### 普通代码块

````
```
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```
````

渲染后:

```
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```

#### 高亮代码块

标记代码的语言类型, 即可高亮显示代码块, 比如: `go`, `python`, `java`, `json`等等, 针对普通文本, 建议使用 `text` 标记.

````
```json
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```
````

渲染后:

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```

> 支持的语言类型列表可参考:
> [https://markdown.land/markdown-code-block](https://markdown.land/markdown-code-block)

### 行内代码块

```text
Use `print("Hello, world!")` to print a message to the screen.
```

渲染后:

Use `print("Hello, world!")` to print a message to the screen.

## 表格

左对齐、居中、右对齐.

```text
| Syntax      | Description | Test Text     |
| ---        |    :----:   |          ---: |
| Header      | Title       | Here's this   |
| Paragraph   | Text        | And more      |
```

| Syntax    | Description |   Test Text |
| :-------- | :---------: | ----------: |
| Header    |    Title    | Here's this |
| Paragraph |    Text     |    And more |

## 有序和无序列表

列表中如果有段落内容, 为了和列表对齐, 段落内容需要以 4 个空格或 1 个 tab 开始.

列表中如果有代码块, 代码块的每一行需要使用 8 个空格或 2 个 tab 开始, 这样才能与列表对齐. 代码块可以加一对三个反引号括起来, 也可以不加.

```text
1. Open the file.

  To add another element in a list while preserving the continuity of the list, indent the element four spaces or one tab.

2. Find the following code block on line 21:

        <html>
          <head>
            <title>Test</title>
          </head>

3. Update the title to match the name of your website.
    - First item
      1. foo
      2. bar
    - Second item
    - Third item
      - Indented item
      - Indented item
    - Fourth item
```

1.  Open the file.

    To add another element in a list while preserving the continuity of the list, indent the element four spaces or one tab.

2.  Find the following code block on line 21:

    ```html
    <html>
      <head>
        <title>Test</title>
      </head>
    </html>
    ```

3.  Update the title to match the name of your website.
    - Second item
      1. foo
      2. bar
    - Third item
      - Indented item
      - Indented item
    - Fourth item

## 任务列表

```text
- [x] Write the press release
- [ ] Update the website
- [ ] Contact the media
```

- [x] Write the press release
- [ ] Update the website
- [ ] Contact the media

## 定义列表

```text
First Term
: This is the definition of the first term.

Second Term
: This is one definition of the second term.
: This is another definition of the second term.
```

First Term
: This is the definition of the first term.

Second Term
: This is one definition of the second term.
: This is another definition of the second term.

## 标题等级

```text
# Heading level 1
## Heading level 2
### Heading level 3
#### Heading level 4
##### Heading level 5
###### Heading level 6
```

等级 1 和等级 2 可替换为:

```text
Heading level 1
===============

Heading level 2
---------------
```

## 使用 `标题ID` 链接到标题

```text
[My Great Heading](#custom-id)

### My Great Heading {#custom-id}
```

[My Great Heading](#custom-id)

### My Great Heading {#custom-id}

## 换行

```text
First line with two spaces after.此处紧邻2个空格
And the next line.

或

First line with the HTML tag after.<br>
And the next line.
```

First line with two spaces after.<br>
And the next line.

## 缩进

```text
&nbsp;&nbsp;&nbsp;&nbsp;This is the first sentence of my indented paragraph.

This is the second sentence of the paragraph.
```

&nbsp;&nbsp;&nbsp;&nbsp;This is the first sentence of my indented paragraph.

This is the second sentence of the paragraph.

## 块引用

```text
> #### The quarterly results look great!
>
> - Revenue was off the chart.
> - Profits were higher than ever.
>
>  *Everything* is going according to **plan**.
```

> #### The quarterly results look great!
>
> - Revenue was off the chart.
> - Profits were higher than ever.
>
> _Everything_ is going according to **plan**.

## 转义反引号

### 转义单个反引号

```text
`` Use `code` in your Markdown file. ``
```

渲染后:

`` Use `code` in your Markdown file. ``

### 转义更多反引号

`````
````
```
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```
````
`````

渲染后:

````
```
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```
````

## 超链接

```sh
# 普通超链接
My favorite search engine is [Duck Duck Go](https://duckduckgo.com).

# 为超链接增加hover提示
My favorite search engine is [Duck Duck Go](https://duckduckgo.com "The best search engine for privacy").

# 让url或mail地址可点击
<https://www.markdownguide.org>
<fake@example.com>

# 在新窗口中打开超链接
<a href="https://www.markdownguide.org" target="_blank">Learn Markdown!</a>

# 禁用超链接
`http://www.example.com`
```

My favorite search engine is [Duck Duck Go](https://duckduckgo.com).

My favorite search engine is [Duck Duck Go](https://duckduckgo.com "The best search engine for privacy").

<https://www.markdownguide.org>

<fake@example.com>

<a href="https://www.markdownguide.org" target="_blank">Learn Markdown!</a>

`http://www.example.com`

## 图片和图片超链接

```sh
# 显示图片
![The San Juan Mountains are beautiful!](https://mdg.imgix.net/assets/images/san-juan-mountains.jpg "San Juan Mountains")

# 图片超链接, 点击图片跳转到某网址
[![An old rock in the desert](https://mdg.imgix.net/assets/images/shiprock.jpg)](https://www.flickr.com/photos/beaurogers/31833779864/in/photolist-Qv3rFw-34mt9F-a9Cmfy-5Ha3Zi-9msKdv-o3hgjr-hWpUte-4WMsJ1-KUQ8N-deshUb-vssBD-6CQci6-8AFCiD-zsJWT-nNfsgB-dPDwZJ-bn9JGn-5HtSXY-6CUhAL-a4UTXB-ugPum-KUPSo-fBLNm-6CUmpy-4WMsc9-8a7D3T-83KJev-6CQ2bK-nNusHJ-a78rQH-nw3NvT-7aq2qf-8wwBso-3nNceh-ugSKP-4mh4kh-bbeeqH-a7biME-q3PtTf-brFpgb-cg38zw-bXMZc-nJPELD-f58Lmo-bXMYG-bz8AAi-bxNtNT-bXMYi-bXMY6-bXMYv)
```

![The San Juan Mountains are beautiful!](https://mdg.imgix.net/assets/images/san-juan-mountains.jpg "San Juan Mountains")

[![An old rock in the desert](https://mdg.imgix.net/assets/images/shiprock.jpg)](https://www.flickr.com/photos/beaurogers/31833779864/in/photolist-Qv3rFw-34mt9F-a9Cmfy-5Ha3Zi-9msKdv-o3hgjr-hWpUte-4WMsJ1-KUQ8N-deshUb-vssBD-6CQci6-8AFCiD-zsJWT-nNfsgB-dPDwZJ-bn9JGn-5HtSXY-6CUhAL-a4UTXB-ugPum-KUPSo-fBLNm-6CUmpy-4WMsc9-8a7D3T-83KJev-6CQ2bK-nNusHJ-a78rQH-nw3NvT-7aq2qf-8wwBso-3nNceh-ugSKP-4mh4kh-bbeeqH-a7biME-q3PtTf-brFpgb-cg38zw-bXMZc-nJPELD-f58Lmo-bXMYG-bz8AAi-bxNtNT-bXMYi-bXMY6-bXMYv)

## 给 Markdown 加注释

注释内容不会渲染. 语法: `[comment]: #`

```text
Here's a paragraph that will be visible.

[This is a comment that will be hidden.]: #

And here's another paragraph that's visible.
```

Here's a paragraph that will be visible.

[this is a comment that will be hidden.]: #

And here's another paragraph that's visible.

## 提示

\> \:warning\: \*\*Warning:\*\* Do not push the big red button.

\> \:memo\: \*\*Note:\*\* Sunrises are beautiful.

\> \:bulb\: \*\*Tip:\*\* Remember to appreciate the little things in life.

渲染后:

> :warning: **Warning:** Do not push the big red button.

> :memo: **Note:** Sunrises are beautiful.

> :bulb: **Tip:** Remember to appreciate the little things in life.

## 打印特殊符号

```text
Copyright (©) — &copy;
Registered trademark (®) — &reg;
Trademark (™) — &trade;
Euro (€) — &euro;
Left arrow (←) — &larr;
Up arrow (↑) — &uarr;
Right arrow (→) — &rarr;
Down arrow (↓) — &darr;
Degree (°) — &#176;
Pi (π) — &#960;
```

查看更多[特殊符号](https://en.wikipedia.org/wiki/List_of_XML_and_HTML_character_entity_references).

> NOTE: 需要你的 Markdown 编辑器支持 HTML.

## 使用 CSS

```text
<p style="text-align:center">Center this text</p>

Not all Markdown applications provide <span style="color:red">CSS</span> support.
```

<p style="text-align:center">Center this text</p>

Not all Markdown applications provide <span style="color:red">CSS</span> support.

> NOTE: 对于不支持 CSS 的 markdown 编辑器, 以上写法无效.

## Markdown 编辑器

<a href="https://www.markdownguide.org/tools/" target="_blank">点击查看 Markdown 编辑器列表</a>

## 参考

> https://github.com/mattcone/markdown-guide
>
> https://www.markdownguide.org/
>
> https://markdown.land/markdown-code-block
