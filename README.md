# jszip-in-vue

本文主要介绍jszip在vue中如何使用

本文涉及到Promise对象和async函数的使用，建议先提前去了解一下

关于jszip的使用官方文档已经介绍的非常详细了，而且也有示例：https://stuk.github.io/jszip/documentation/examples.html

我这里主要是结合项目的需求然后抽离出来的demo，主要是对图片的加压和解压

## 解压

使用.loadAsync(data)可以加载zip文件，data必须是二进制流，我这里是使用html原生的input上传文件来获取文件流，如果需要读取本地文件可以使用JSZipUtils.getBinaryContent(zipPath, callback)
```
JSZipUtils.getBinaryContent('path/to/content.zip', function(err, data) {
    if(err) {
        throw err; // or handle err
    }

    JSZip.loadAsync(data).then(function () {
        // ...
    });
});

// or, with promises:

new JSZip.external.Promise(function (resolve, reject) {
    JSZipUtils.getBinaryContent('path/to/content.zip', function(err, data) {
        if (err) {
            reject(err);
        } else {
            resolve(data);
        }
    });
}).then(function (data) {
    return JSZip.loadAsync(data);
})
.then(...)
```

使用.file(name).async()读取文件里面的内容。
```
zip.file("hello.txt").async("string").then(function (data) {
  // 以字符串形式输出文本内容
});

if (JSZip.support.uint8array) {
  zip.file("hello.txt").async("uint8array").then(function (data) {
    // 使用二进制数组存储文本内容
  });
}
```
以上的方法都是Promise实例，可以用then方法指定resolved状态的回调函数。

本示例中通过上传文件获取文件并将其解压将文件内容显示在页面上
``` js
const zipFile = e.target.files[0]
const jszip = new JSZip()
jszip.loadAsync(zipFile).then((zip) => { // 读取zip
    for (let key in zip.files) { // 判断是否是目录
        if (!zip.files[key].dir) {
            if (/\.(png|jpg|jpeg|gif)$/.test(zip.files[key].name)) { // 判断是否是图片格式
                let base = zip.file(zip.files[key].name).async(
                    'base64') // 将图片转化为base64格式
                base.then(res => {
                    this.dataList.push({
                        fileName: zip.files[key].name,
                        type: 'img',
                        content: `data:image/png;base64,${res}`
                    })
                })
            }
            if (/\.(txt)$/.test(zip.files[key].name)) { // 判断是否是文本文件
                let base = zip.file(zip.files[key].name).async(
                    'string') // 以字符串形式输出文本内容
                base.then(res => {
                    this.dataList.push({
                        fileName: zip.files[key].name,
                        type: 'text',
                        content: res
                    })
                })
            }
        }
    }
})
```

## 加压
使用.file(name, content)将内容写进文件
```
zip.file("Hello.txt", "Hello world\n");
```
使用.generateAsync(option)将文件转换为流文件，并使用FileSaver的saveAs(content, fileaname)将文件保存到本地
```
zip.generateAsync({type:"blob"})
.then(function (blob) {
    saveAs(blob, "hello.zip");
});
```

本示例主要是对图片进行加压。先将图片转换为base64格式（使用canvas），再将其转换为二进制流进行加压。因为是使用img.onload异步加载图片，必须等待图片加载完后才能将其进行加压，所以需要用async函数将异步转换为同步。
``` js
/**
 *
 * @param url 图片路径
 * @param ext 图片格式
 * @param callback 结果回调
 */
async getUrlBase64(url, ext) {
    return new Promise((resolve, reject) => {
        var canvas = document.createElement('canvas') // 创建canvas DOM元素
        var ctx = canvas.getContext('2d')
        var img = new Image()
        img.crossOrigin = 'Anonymous' // 处理跨域
        img.src = url
        img.onload = () => {
            canvas.width = img.width // 指定画板的高度,自定义
            canvas.height = img.height // 指定画板的宽度，自定义
            ctx.drawImage(img, 0, 0) // 参数可自定义
            var dataURL = canvas.toDataURL('image/' + ext)
            resolve(dataURL) // 回调函数获取Base64编码
            canvas = null
        }
    })
},
async exportZip() {
    const imgList = this.imgList
    const proList = []
    const zip = new JSZip()
    const cache = {}
    await imgList.forEach(item => { // 等待所有图片转换完成
        const pro = this.getUrlBase64(item, '.jpg').then(base64 => {
            const fileName = item.replace(/(.*\/)*([^.]+)/i,"$2")
            zip.file(fileName, base64.substring(base64.indexOf(',') + 1), {
                base64: true
            })
            cache[fileName] = base64
        })
        proList.push(pro)
    })
    Promise.all(proList).then(res => {
        zip.generateAsync({
            type: 'blob'
        }).then(content => { // 生成二进制流
            saveAs(content, 'images.zip') // 利用file-saver保存文件
        })
    })
}
```
***
以上读取和保存文件的方法只使用于在浏览器中使用，如果需要在node.js中读取和保存文件的话可以使用node.js内置模块fs。

## GitHub地址
https://github.com/13660539818/jszip-in-vue

## 参考文档
https://blog.csdn.net/qq_32858649/article/details/88759454

https://segmentfault.com/a/1190000018234223
