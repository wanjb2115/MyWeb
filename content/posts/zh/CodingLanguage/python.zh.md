---
title: "编程时用过的python小技巧"
date: 2021-09-26T19:14:44+08:00
draft: true
tags: ["python"]
categories: ["编程语言"]
Summary: "编程时用过的python小技巧"
---

### 设置进程名称

```python
import sys
import setproctitle

reload(sys)
sys.setdefaultencoding('utf-8')
setproctitle.setproctitle('ProgramName')
```


### os获取~目录

```python
import os

home_dir = os.path.expanduser('~')
home_dir = os.path.join(home_dir, "wanjb")
```

### logging配置日志文件与日志大小

```python
import logging, os
from logging.handlers import RotatingFileHandler

logFile = os.path.abspath(__file__).replace(".py", ".log")
logger = logging.getLogger(logFile)

# consoleformat = logging.Formatter("[%(asctime)s] %(levelname)-8s %(threadName)s - %(message)s", "%m-%d %H:%M:%S")
# consolehandler = logging.StreamHandler(sys.stdout)
# consolehandler.setLevel(logging.INFO)
# consolehandler.setFormatter(consoleformat)
# logger.addHandler(consolehandler)

# fileformat = logging.Formatter("[%(asctime)s][%(filename)s %(lineno)d] %(levelname)-8s  %(threadName)s - %(message)s", "%m-%d %H:%M:%S")
# filehandler = logging.FileHandler(logFile, mode="w")
# filehandler.setLevel(logging.INFO)
# filehandler.setFormatter(fileformat)
# logger.addHandler(filehandler)

fileSize = RotatingFileHandler(logFile, mode='a', maxBytes=10*1024*1024, backupCount=2, encoding='utf-8', delay=0)

logger.addHandler(fileSize)
```

