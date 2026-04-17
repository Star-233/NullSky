---
tags:
  - CTF
---
zip压缩包的**本地文件头(Local File Header, LFH)**和**中央目录结构(Central Directory Structure, CDS)**中的**通用目的标志位**用于描述多个信息。

**通用目的标志位**中含有多个标志位，其中加密标志位代表是否加密，值是 `0` 或 `1` 。

压缩包里的每个文件都有自己的LFH和CDS，通常压缩包只看CDS。

010editor的模板里，`struct ZIPFILERECORD` 和 `struct ZIPDIRENTRY` 一般是LFH和CDS。 
