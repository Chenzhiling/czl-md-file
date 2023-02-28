# 将Springboot的MultipartFile文件上次到Hadoop文件系统

## 1. 简介

`MultipartFile`是`SpringMvc`提供简化上传操作的工具类。在由`Springboot`搭建的后台项目中，在页面上传文件，后台接口一般用的都是这个类。

这篇文章介绍如何将`MultiparFile`文件上传到`Hadoop`文件系统上

## 2. 实现

`MultipartFile`的`getInputStream`方法，获得输入流，随后转换成`hadoop`的`FSDataOutputStream`输出流，将文件上传到`hadoop`。

最后记得`flusu`和`close`输入输出流。

## 3. 代码

```java
private void uploadMultiPartFileToHdfs(MultipartFile file,String uploadPath) throws IOException {
        InputStream inputStream = null;
        FSDataOutputStream fsDataOutputStream = null;
        try {
            inputStream = file.getInputStream();
            FileSystem hdfs = FileSystem.get(new URI(""), new Configuration());
            Path desPath = new Path(uploadPath + "/" + file.getOriginalFilename());
            fsDataOutputStream = hdfs.create(desPath);
            byte[] b = new byte[4096];
            int read;
            while ((read=inputStream.read(b)) > 0) {
                fsDataOutputStream.write(b, 0, read);
            }
            fsDataOutputStream.flush();
        } catch (IOException exception) {
            exception.printStackTrace();
        } finally {
            if (null != inputStream) inputStream.close();
            if (null != fsDataOutputStream) fsDataOutputStream.close();
        }
    }
```

