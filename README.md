# Photo Shooting Date

Photo Shooting Date 是一个用于批量修改图片拍摄时间的浏览器工具。它可以为 JPG/JPEG 图片写入 EXIF 拍摄时间，也支持为 PNG 图片写入 `eXIf` 元数据块，适合整理相机、手机、扫描件或迁移后的照片时间信息。

项目在浏览器本地处理图片，不需要上传图片到服务器。

## 功能

- 批量选择或拖拽上传图片
- 支持 JPG、JPEG、PNG 格式
- 使用文件头识别图片类型，避免只依赖扩展名或浏览器 MIME 类型
- 将选择的日期时间写入图片拍摄时间元数据
- 批量打包下载处理后的图片
- 对不支持的图片格式进行拦截和错误提示

## 使用场景

- 批量修正照片拍摄时间
- 给扫描件、导入照片或缺失时间信息的图片补充拍摄时间
- 在迁移相册、整理素材、归档图片前统一元数据
- 为图片管理软件提供更准确的时间排序依据

## 支持格式

| 格式 | 处理方式 |
| --- | --- |
| JPG/JPEG | 写入 EXIF `DateTimeOriginal` |
| PNG | 写入 PNG `eXIf` chunk |

注意：不同图片查看器、相册软件对 PNG `eXIf` 元数据的支持程度不同。如果目标软件不读取 PNG 的 `eXIf` chunk，可能无法显示写入后的拍摄时间。

## 本地开发

安装依赖：

```bash
npm install
```

启动开发服务：

```bash
npm run serve
```

构建生产版本：

```bash
npm run build
```

运行代码检查：

```bash
npm run lint
```

## 技术栈

- Vue 3
- Element Plus
- JSZip
- piexifjs

## 维护计划

- 增加更多图片元数据字段支持
- 补充图片处理逻辑的自动化测试
- 改进 PNG 元数据兼容性说明
- 增加在线演示和使用截图

## License

MIT
