---
title: Neovim自定义代码片段
date: 2025-10-09 23:43:28
tags:
  - nvim
  - snippets
categories:
  - Neovim
---

用[LuaSnip](https://github.com/L3MON4D3/LuaSnip)可以实现自定义代码片段,
从而避免一些重复的typing.

---

起因是这样的, 很多开源的代码中, 文件头部都有统一的关于License的注释, 比如:

```txt
SPDX-License-Identifier: ...
```

还有一些Doxygen的文件头注释:

```txt
/*! 
 * @file
 * @author Firstname Lastname
 * @version 1.0
 * @copyright Copyright (c) 2025
 */
```

这些代码片段都可以通过预先定义snippets来快速插入. 可以参考这个commit的设置:
[qiujiandong/kickstart.nvim:99bf05b](https://github.com/qiujiandong/kickstart.nvim/commit/99bf05bcbc168d214e33cc98e06566299969d622)

```lua
return {
  -- C file header snippet
  s(
    'header_apache',
    fmt(
      [[
/**
 * @file {project}.c
 * @brief {description}
 * @author Jiandong Qiu
 * @date {}
 *
 * Copyright (c) {} Jiandong Qiu
 * SPDX-License-Identifier: Apache-2.0
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

]],
      {
        project = i(1, 'ProjectName'),
        description = i(2, 'Short description'),
        f(function()
          return os.date '%Y-%m-%d'
        end),
        os.date '%Y',
      }
    )
  ),
}
```

效果就是只需要输入`header_apache`, 就会自动插入上述的代码片段.

如果之后遇到开发中经常写的代码片段, 也都可以用类似的方式写一段snippets, 从而提高效率.
