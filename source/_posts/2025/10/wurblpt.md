---
title: 用wurblpt仿真ToF传感器
date: 2025-10-12 23:21:36
tags:
  - sensor
  - sim
  - wurblpt
categories:
  - sensor
---

[WurblPT](https://github.com/marlam/wurblpt)

最开始是从这篇论文[Simulation of Time-of-Flight Sensors for Evaluation of Chip Layout Variants](https://ieeexplore.ieee.org/document/7054461)
入手的, WurblPT有一个example用来仿真这个论文里的例子.

仓库依赖`tgd`, 可以先把这个仓库[tgd](https://github.com/marlam/tgd) clone一下,
先编译这个

``` bash
cd tgd
cmake -B build 
cmake -B build -DCMAKE_INSTALL_PREFIX=$HOME/.local -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_BUILD_TYPE=Release .
cd build
sudo checkinstall
```

值得注意的有两个地方:

1. 一定要编译Release模式, Release模式跑起来真的快很多.
2. 这种用源码编译的库我一般都放在自己的`.local`目录, 那样不容易污染系统环境.
3. 用`checkinstall`安装可以方便后续卸载, 但就是需要用到管理员权限,
卸载的时候也是通过dpkg卸载. 在安装的时候要记得看清楚安装的库的名字.

安装完`tgd`之后就可以安装`libwurblpt`了, 安装的命令也是一样的.

``` bash
cd wurblpt/libwurblpt
cmake -B build -DCMAKE_INSTALL_PREFIX=$HOME/.local -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_BUILD_TYPE=Release .
cd build
sudo checkinstall
```

最后再编译`wurblpt-tof-example`, 然后就可以跑ToF的仿真了.

```bash
cd wurblpt/wurblpt-tof-example
cmake -B build -DCMAKE_INSTALL_PREFIX=$HOME/.local -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_BUILD_TYPE=Release .
cd build
make
./wurblpt-tof-example
```

仿真生成的都是`.tgd`后缀的数据, 可以用`qv`查看, 这个我也是从仓库的issue看到的,
[wurblpt#1](https://github.com/marlam/wurblpt/issues/1)
我是通过`flatpak`安装的`qv`, 可以把要看的一个系列图片放在一个文件夹里,
然后用`qv`查看整个文件夹, 然后可以用方向键的左右来看每帧的结果.

```bash
flatpak run de.marlam.qv result
```

如何将距离像转换为三维点云视图? [这里](https://marlam.de/wurblpt/features/simulation-of-time-of-flight-sensors/)
只说了可以用距离像生成, 但是没有说具体怎么做. 而且`tgd`格式在python里的支持不是很好,
没法直接读取. `tgd`格式可以转`tiff`格式:  
比如像这样可以把`resutl.tgd`中通道0的数据转换称`tiff`格式.

```bash
tgd convert -c 0 result.tgd result.tiff
```

然后`tiff`格式的数据在python中就可以比较方便处理了. 利用`open3d`可以生成3D点云.

```python
import tifffile as tiff
import open3d as o3d
import numpy as np
import matplotlib.pyplot as plt

Z = tiff.imread("out.tiff")
Z = 300 - Z * 100
x, y = np.meshgrid(np.arange(Z.shape[1]), np.arange(Z.shape[0])[::-1])

points = np.stack((x, y, Z), axis=-1).reshape(-1, 3)

norm = (Z - Z.min()) / (Z.max() + 50 - Z.min())
colors = plt.cm.jet(norm).reshape(-1, 4)[:, :3]

pcd = o3d.geometry.PointCloud()
pcd.points = o3d.utility.Vector3dVector(points)
pcd.colors = o3d.utility.Vector3dVector(colors)

min_bound = points.min(axis=0)
max_bound = points.max(axis=0)

bbox_points = np.array([
    [min_bound[0], min_bound[1], min_bound[2]],
    [max_bound[0], min_bound[1], min_bound[2]],
    [max_bound[0], max_bound[1], min_bound[2]],
    [min_bound[0], max_bound[1], min_bound[2]],
    [min_bound[0], min_bound[1], max_bound[2]],
    [max_bound[0], min_bound[1], max_bound[2]],
    [max_bound[0], max_bound[1], max_bound[2]],
    [min_bound[0], max_bound[1], max_bound[2]],
])

lines = [
    [0,1],[1,2],[2,3],[3,0],
    [4,5],[5,6],[6,7],[7,4],
    [0,4],[1,5],[2,6],[3,7] 
]

colors_lines = [[0,0,0] for _ in lines]

line_set = o3d.geometry.LineSet()
line_set.points = o3d.utility.Vector3dVector(bbox_points)
line_set.lines = o3d.utility.Vector2iVector(lines)
line_set.colors = o3d.utility.Vector3dVector(colors_lines)

render_option = o3d.visualization.RenderOption()
render_option.background_color = np.array([1.0, 1.0, 1.0])

o3d.visualization.draw_geometries([pcd, line_set])
```


