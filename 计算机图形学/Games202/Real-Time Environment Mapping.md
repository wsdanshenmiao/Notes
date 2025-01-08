# Real-Time Environment Mapping

## Environment Lighting

使用一张图来记录各个方向的光照，该光照来自无限远，可通过Spherical Map和Cube Map来储存。

## Shading form Environment Light(Image-Based Lighting)

使用环境光在不考虑阴影的情况下计算Shading，需要解渲染方程<img src="https://img2024.cnblogs.com/blog/3406761/202410/3406761-20241016121336522-1872808055.png" style="zoom: 33%;" />

通用