
# 大文件传输与断点续传实现（极简Demo:React\+Node.js）


## 简述


使用React前端和Node.js后端实现大文件传输和断点续传的功能。通过分片上传技术，可以有效地解决网络不稳定带来的传输中断问题。


## 文章内容


### 前端实现（React）


首先，您需要在前端项目中安装`axios`、`spark-md5`库以处理HTTP请求。可以使用以下命令：



```
npm i axios spark-md5

```

以下是实现大文件上传的React组件代码：



```
import axios from 'axios';
import { useRef, useState } from 'react';

// 定义文件分片大小（例如 5MB）
const CHUNK_SIZE = 5 * 1024 * 1024;

/** 解析当前页面功能 */
export default function FileUploader() {
  const [file, setFile] = useState(null); // 用于存储用户选择的文件
  const [uploadProgress, setUploadProgress] = useState(0); // 上传进度
  const uploading = useRef(false); // 用于防止重复触发上传逻辑

  // 当用户选择文件时触发
  const handleFileChange = (e) => {
    setFile(e.target.files[0]); // 将选择的文件存入状态
    setUploadProgress(0); // 重置上传进度
  };

  // 计算文件的唯一标识 (哈希)
  const calculateFileHash = async (file) => {
    return new Promise((resolve) => {
      const reader = new FileReader();
      reader.onload = (e) => {
        const sparkMD5 = require('spark-md5');
        const hash = sparkMD5.ArrayBuffer.hash(e.target.result);
        resolve(hash);
      };
      reader.readAsArrayBuffer(file);
    });
  };

  // 开始文件上传
  const handleUpload = async () => {
    if (!file || uploading.current) return; // 如果未选择文件或正在上传，则直接返回

    uploading.current = true; // 标记为正在上传
    const fileHash = await calculateFileHash(file); // 获取文件的唯一标识（哈希值）
    console.log('fileHash', fileHash);
    const totalChunks = Math.ceil(file.size / CHUNK_SIZE); // 计算文件分片总数
    // 检查哪些分片已经上传
    const { data: uploadedChunks } = await axios.post(
      'http://localhost:5000/check',
      {
        fileName: file.name,
        fileHash,
      },
    );

    // 上传未完成的分片
    for (let chunkIndex = 0; chunkIndex < totalChunks; chunkIndex++) {
      if (uploadedChunks?.includes(chunkIndex)) {
        console.log('跳过chunkIndx', chunkIndex);
        setUploadProgress(((chunkIndex + 1) / totalChunks) * 100); // 更新进度
        continue; // 跳过已经上传的分片
      }
      console.log('上传chunkIndx', chunkIndex);
      // 创建当前分片
      const start = chunkIndex * CHUNK_SIZE; // 分片起始字节
      const end = Math.min(file.size, start + CHUNK_SIZE); // 分片结束字节
      const chunk = file.slice(start, end); // 获取分片

      // 上传分片
      const formData = new FormData();
      formData.append('chunk', chunk); // 当前分片
      formData.append('fileName', file.name); // 文件名
      formData.append('fileHash', fileHash); // 文件唯一标识
      formData.append('chunkIndex', chunkIndex); // 分片索引

      await axios.post(
        `http://localhost:5000/upload?fileHash=${fileHash}&chunkIndex=${chunkIndex}&fileName=${file.name}`,
        formData,
        {
          onUploadProgress: (progressEvent) => {
            const progress =
              ((chunkIndex + progressEvent.loaded / progressEvent.total) /
                totalChunks) *
              100;
            setUploadProgress(progress); // 实时更新上传进度
          },
        },
      );
    }

    // 通知服务端合并分片
    await axios.post('http://localhost:5000/merge', {
      fileName: file.name,
      fileHash,
      totalChunks,
    });

    alert('上传成功！');
    uploading.current = false; // 标记上传完成
  };

  return (
    <div style={{ padding: '20px' }}>
      <p>大文件上传（支持断点续传）p>
      <input type="file" onChange={handleFileChange} />
      <button onClick={handleUpload}>提交上传文件button>
      <div style={{ marginTop: '20px' }}>
        <progress value={uploadProgress} max="100" />
        <div>上传进度：{uploadProgress.toFixed(2)}%div>
      div>
    div>
  );
}


```

### 后端实现（Node.js）


在后端，您需要安装以下依赖：`multer`、`fs-extra`、`express`、`cors`、`body-parser`。可以使用以下命令：



```
npm i multer fs-extra express cors body-parser

```


> **注意**: 这些包应用于生产环境和开发环境。


以下是Node.js服务器的实现代码：



```
// 文件：server.js
const express = require("express");
const multer = require("multer");
const fs = require("fs");
const bodyParser = require("body-parser");
const path = require("path");
const cors = require("cors");
const app = express();
const uploadDir = path.join(__dirname, "uploads"); // 上传目录

// 确保上传目录存在
if (!fs.existsSync(uploadDir)) {
  fs.mkdirSync(uploadDir);
}

app.use(cors()); // 允许跨域请求
app.use(bodyParser.json());
app.use(express.json()); // 解析 JSON 请求体

// 检查已上传的分片
app.post("/check", (req, res) => {
  const { fileHash } = req.body;
  console.log("fileHash check",fileHash)

  const fileChunkDir = path.join(uploadDir, fileHash); // 分片存储目录
  if (!fs.existsSync(fileChunkDir)) {
    return res.json([]); // 如果目录不存在，返回空数组
  }

  // 返回已上传的分片索引
  const uploadedChunks = fs.readdirSync(fileChunkDir).map((chunk) => {
    return parseInt(chunk.split("-")[1]); // 提取分片索引
  });
  res.json(uploadedChunks);
});


// 设置 multer 中间件，用于处理文件上传
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    const fileHash = req.query.fileHash; // 从查询参数获取 fileHash
    const chunkDir = path.join(uploadDir, fileHash);
    // 确保切片目录存在
    if (!fs.existsSync(chunkDir)) {
      fs.mkdirSync(chunkDir, { recursive: true });
    }
    cb(null, chunkDir);
  },
  filename: (req, file, cb) => {
    const { chunkIndex } = req.query;
    cb(null, `chunk-${chunkIndex}`);
  },
});

const upload = multer({ storage:storage });

// 上传文件分片
app.post("/upload", upload.single("chunk"), (req, res) => {
    const { fileHash } = req.body;
    res.status(200).send("分片上传成功");
});


// 合并分片
app.post("/merge", (req, res) => {
  const { fileName, fileHash, totalChunks } = req.body;
  console.log("fileName",req.body)
  const fileChunkDir = path.join(uploadDir, fileHash);
  const filePath = path.join(uploadDir, fileName);

  // 创建可写流用于最终合并文件
  const writeStream = fs.createWriteStream(filePath);

  for (let i = 0; i < totalChunks; i++) {
    const chunkPath = path.join(fileChunkDir, `chunk-${i}`);
    const data = fs.readFileSync(chunkPath); // 读取分片
    writeStream.write(data); // 写入最终文件
    // fs.unlinkSync(chunkPath); // 删除分片文件--留下来，可以看上传记录
  }

  writeStream.end(); // 关闭流
//   fs.rmdirSync(fileChunkDir); // 删除分片目录--留下来，可以看上传记录
  res.send("文件合并完成");
});

app.listen(5000, () => {
  console.log("服务器已启动：http://localhost:5000");
});

```

### 总结


通过上述代码，您可以实现大文件的分片上传和断点续传功能。这种方法不仅提高了上传的可靠性，还能有效应对网络的不稳定性。希望这篇文章对您有所帮助！


 本博客参考[cmespeed楚门加速器](https://77yingba.com)。转载请注明出处！
