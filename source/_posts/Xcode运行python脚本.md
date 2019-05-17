---
title: Xcode运行python脚本
date: 2019-05-16 21:03:30
toc: true
description: Xcode可以允许我们使用一些脚本进行编译前的自动化工作,通常使用shell脚本,但是很多人可能会和我一样对python脚本更有兴趣,那么怎样使用Xcode运行python脚本呢?
---

## Xcode运行python脚本
Xcode可以允许我们使用一些脚本进行编译前的自动化工作,通常使用shell脚本,但是很多人可能会和我一样对python脚本更有兴趣,那么怎样使用Xcode运行python脚本呢?

我们知道shell脚本可以直接访问xcode的环境变量,但是python脚本却不行.

打开TARGETS->Build Phases->Run Script
将下面shell脚本复制进去,或者提取到外部shell脚本中执行.
注意这里还是实用的shell `/bin/sh`

```
function callPythonScript() {
    # 将GCC_PREPROCESSOR_DEFINITIONS参数传递到puthon的<sys.argv>里.
    python $1 ${GCC_PREPROCESSOR_DEFINITIONS};

    # 错误判断.
    return_code=$?
    if [ $return_code -ne 0 ]; then
    exit 1;
    fi
}

# 使用上面的callPythonScript()函数执行python脚本.
cd $SRCROOT 
cd ..
callPythonScript ./ptScript.py

exit 0;

```

ptScript.py

```py
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import os, sys
print os.environ['PRODUCT_NAME']

# 控制台输出
# >> DemoProject
```


