# 2024/09/27
在OpenSceneGraph源码基础上做小修改。

## Osg读取ply二进制文件报错问题
osgDB：读取纹理等资源，但默认支持的图片格式不包含jpg和png，需要编译时加上第三方的库。默认支持bmp。

背景：ubuntu 20.04 + Osg 3.7 + ply二进制文件，osg编译时选上了ply插件，是有ply对应的plugin的。

使用osg的`readNodeFile`直接读取ply二进制文件
```c++
osg::ref_ptr<osg::Node> pMeshNode = osgDB::readNodeFile(file_name);
```
报错
```c++
Unable to read PLY file, an exception occurred:  get_binary_item: bad type = 0
```

ply文件头如下
```
ply
format binary_little_endian 1.0
comment TextureFile scene_dense_mesh_06-03-000.png
element vertex 15
property float32 x
property float32 y
property float32 z
element face 20
property list uint8 uint32 vertex_indices
property list uint8 float32 texcoord
end_header
```

把osg中ply相关的单独拿出来调试，发现报错出现在`VertexData::readTriangles`读取list的时候`prop->external_type`为０，在`get_binary_item`中找不到对应的类型。

而该类型在读取文件头时设置
```c++
// plyfile.cpp
  if (equal_strings (words[1], "list")) {       /* is a list */
    prop->count_external = get_prop_type (words[2]);
    prop->external_type = get_prop_type (words[3]);
```    
其中`get_prop_type`中`type_names`没有`uint32`导致后续读取数据时报错

## Osg读取ply二进制文件显示没有纹理
类型修改匹配后加载模型未报错，但纹理没有正确显示。

调试发现`_texcoord`为空，查看代码逻辑发现Osg只在读取顶点时判断是否有纹理坐标信息，读取面时只处理顶点索引。即，Osg不支持同一顶点不同纹理坐标的模型。
