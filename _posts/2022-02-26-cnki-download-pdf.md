---
title: "怎样在CNKI上下载学位论文的PDF版"
math: true
categories: 
  - "Life"
tags: 
  - "CNKI"
---

CNKI是包含有中国大部分学位论文（除了PKU）的数据库，一直以来着力推广CAJ文件格式（让你下它那个CAJ Viewer），而不提供学位论文的PDF版下载，非常难用（有些文献管理工具打不开CAJ）。

其实CNKI并不是没有PDF版，只是隐藏了而已。如果使用Chrome内核浏览器整个插件是最方便的，如果不方面安装的话可以参考下面两个方法。

## 1. 知网海外版

把论文详情页面的域名从`kns.cnki.net`修改为`oversea.cnki.net`或者`chn.oversea.cnki.net`，可以发现多了一个PDF下载的按钮。

![](/uploads/img/posts/2022-02-26-cnki-download-pdf/domestic-download.png){: width="500" }
_中文版_

![](/uploads/img/posts/2022-02-26-cnki-download-pdf/oversea-download.png){: width="600" }
_海外版_

## 2. 修改下载链接

有些网络条件打不开知网海外版，也可以在论文检索页面的下载链接（链接格式为：`https://kns.cnki.net/KNS8/download?filename=`）末尾加一个参数`&dflag=pdfdown`

![](/uploads/img/posts/2022-02-26-cnki-download-pdf/cnki-paper-download-link.png){: width="400" }
_检索界面的下载按钮_

参考：<https://www.zhihu.com/question/25275044/answer/793372843>
