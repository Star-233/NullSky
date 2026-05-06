https://portswigger.net/web-security/learning-paths/ssrf-attacks/ssrf-attacks-blind-ssrf-vulnerabilities/ssrf/blind/lab-out-of-band-detection

统计数据分析软件会访问 Referer的地址，把 Referer 改成 Burpsuite 里 Collaborator 的地址就行。

![[Pasted image 20260506180021.png]]

---


> [!NOTE] 
> 数据分析软件往往会访问 Referer，其目的可能是
> - 抓取页面的某些内容
> - 防止“引荐来源伪造（Referer Spreading）”垃圾邮件，软件会反向请求该 URL，确认这个页面上是否真的存在指向本站的链接。
> - SEO 与权重分析：检查来源页面的权重、关键词等，用于市场分析。




