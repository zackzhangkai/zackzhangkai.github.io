---
layout:       post
title:        "爬虫之selenium应用"
date:         2017-3-5 12:00:00
categories: document
tag:
  - python
  - 爬虫
---

* content
{:toc}


#### 代码示例

```python
from selenium import webdriver
from selenium.webdriver.common.keys import Keys

driver = webdriver.Firefox()
driver.get("http://www.python.org")
assert "Python" in driver.title
elem = driver.find_element_by_name("q")
elem.clear()
elem.send_keys("pycon")
elem.send_keys(Keys.RETURN)
assert "No results found." not in driver.page_source
driver.close()
```
