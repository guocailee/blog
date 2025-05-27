---
{"dg-publish":true,"permalink":"/Program/Mixed/EPUB格式内容/","noteIcon":"","created":"2024-12-24T10:58:37.300+08:00"}
---

EPUB 是一种开放的电子书格式，其本质是一个包含多个文件（HTML、CSS、图片、XML 等）的压缩包（ZIP 格式）。以下是 EPUB 文件的主要结构和组成部分：

---

## 1. **基础结构**

EPUB 文件通常包含以下几个关键部分：

### 顶层目录结构

```
example.epub
├── mimetype
├── META-INF/
│   └── container.xml
└── OEBPS/ (或内容目录)
    ├── content.opf
    ├── toc.ncx
    ├── chapter1.xhtml
    ├── chapter2.xhtml
    ├── style.css
    └── images/
        └── cover.jpg
```

---

## 2. **关键文件**

### (1) `mimetype`

- 描述 EPUB 文件的 MIME 类型。
- 文件内容：
    
    ```
    application/epub+zip
    ```
    
- **重要**：此文件必须是 EPUB 压缩包的第一个文件，且不能被压缩，否则会导致不兼容。

---

### (2) `META-INF/container.xml`

- 指定 EPUB 的主内容文件位置。
- 结构示例：
    
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <container version="1.0" xmlns="urn:oasis:names:tc:opendocument:xmlns:container">
        <rootfiles>
            <rootfile full-path="OEBPS/content.opf" media-type="application/oebps-package+xml"/>
        </rootfiles>
    </container>
    ```
    

---

### (3) `OEBPS/content.opf` (OPF 文件)

- 是 EPUB 的核心文件，定义电子书的元数据、资源清单和阅读顺序。
- 结构示例：
    
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <package xmlns="http://www.idpf.org/2007/opf" version="3.0" unique-identifier="book-id">
        <metadata>
            <dc:title xmlns:dc="http://purl.org/dc/elements/1.1/">电子书标题</dc:title>
            <dc:creator xmlns:dc="http://purl.org/dc/elements/1.1/">作者</dc:creator>
            <dc:language xmlns:dc="http://purl.org/dc/elements/1.1/">zh</dc:language>
        </metadata>
        <manifest>
            <item id="chapter1" href="chapter1.xhtml" media-type="application/xhtml+xml"/>
            <item id="style" href="style.css" media-type="text/css"/>
            <item id="cover" href="images/cover.jpg" media-type="image/jpeg"/>
        </manifest>
        <spine>
            <itemref idref="chapter1"/>
        </spine>
    </package>
    ```
    

---

### (4) `OEBPS/toc.ncx` (导航文件)

- 描述电子书的目录结构。
- 结构示例：
    
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <ncx xmlns="http://www.daisy.org/z3986/2005/ncx/" version="2005-1">
        <head>
            <meta name="dtb:uid" content="book-id"/>
            <meta name="dtb:depth" content="1"/>
        </head>
        <docTitle>
            <text>电子书标题</text>
        </docTitle>
        <navMap>
            <navPoint id="chapter1" playOrder="1">
                <navLabel>
                    <text>第一章</text>
                </navLabel>
                <content src="chapter1.xhtml"/>
            </navPoint>
        </navMap>
    </ncx>
    ```
    

---

## 3. **内容文件**

### (1) HTML/XHTML 文件

- 章节内容通常以 HTML 或 XHTML 格式存储。
- 示例：
    
    ```html
    <!DOCTYPE html>
    <html xmlns="http://www.w3.org/1999/xhtml">
    <head>
        <title>第一章</title>
        <link rel="stylesheet" type="text/css" href="style.css"/>
    </head>
    <body>
        <h1>第一章</h1>
        <p>这是第一章的内容。</p>
    </body>
    </html>
    ```
    

### (2) CSS 文件

- 样式文件，用于控制章节内容的外观。
- 示例：
    
    ```css
    body {
        font-family: "Arial", sans-serif;
        line-height: 1.6;
    }
    
    h1 {
        color: #333;
    }
    ```
    

### (3) 图片和其他资源

- 图片、音频或视频资源通常存储在子目录下，例如 `images/` 或 `media/`。

---

## 4. **压缩文件**

- 完整的 EPUB 文件是一个 ZIP 压缩包。
- 压缩顺序：
    1. `mimetype` 文件（必须存放在 ZIP 的开头，且不压缩）。
    2. 其他文件按目录结构打包。

---

## 5. **EPUB 版本差异**

- **EPUB 2.0.1**：较旧版本，广泛支持，但功能有限。
- **EPUB 3.x**：支持 HTML5、CSS3、多媒体和交互功能，适合更复杂的电子书。

---

通过上述结构理解，可以更灵活地手动创建 EPUB 文件，或通过代码生成符合规范的电子书。