---
layout: post
title: Java简单实现文件的上传下载
tags: [Java, SpringBoot, SpringData]
---

## 介绍

之所以说简单呢，是因为我把上传的文件直接按二进制的方式存入数据库中了。采用SpringBoot框架，利用SpringData实现持久化操作。

## 实现

### 1.controller层

``` java
    @Autowired
	private MyFileDao myfileDao;
	
	//upload file
	@PostMapping(value = "file")
    @ResponseBody
    public ResponseEntity<?> uploadFile(@RequestParam MultipartFile file) throws IOException {
        //利用对象存储
        Myfile myfile = new Myfile();
        myfile.setFile(file.getBytes())
        
		myfileDao.save(myfile);
        return new ResponseEntity<>(HttpStatus.OK);
    }

	//download file
   	@GetMapping(value = "file")
    @ResponseBody
    public ResponseEntity<?> downloadFile(@RequestParam Long fileId) {
        ResponseEntity responseEntity;
		byte[] fileByte;
        String fileName;
        
        Optional<Myfile> myFileOptional = myFileDao.findById(fileId); 
        if (myFileOptinonal.isPresent()) {
			fileByte = myFileOptional.get().getFileBytes();
            fileName = myFileOptional.get().getFileName();
        }
		
        //设置header这步很关键，告诉浏览器返回的是文件
        HttpHeaders headers = new HttpHeaders();
        headers.add("Content-Disposition", "attachment;filename=" + fileName);

        responseEntity = new ResponseEntity<>(fileByte, headers, HttpStatus.OK);
        return responseEntity;
    }
```

这里简便起见省略了service层

### 2.dao层

``` java
public interface AlarmCsvDao extends CrudRepository<MyFile, Long> {
}
```

### 3.entity

``` java
public class MyFile {

    private Long id;
    private String fileName;
    private byte[] fileBytes;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    
    public String getFileName() {
        return fileName;
    }

    public void setFileName(String fileName) {
        this.fileName = fileName;
    }

    public byte[] getFileBytes() {
        return fileBytes;
    }

    public void setFileBytes(byte[] fileBytes) {
        this.fileBytes = fileBytes;
    }
}
```

## 总结

​	这种将文件存入关系型数据库的方式并不多见，我也只是因为业务需要才采取这种方式的。就这么简单一个过程，其实还有段小插曲，我刚开始做的时候总是想着怎么去处理输入输出流巴拉巴拉一堆，浪费了俩个小时还是云里雾里，直到我起身去倒水的时候猛然醒悟。我在干嘛呀，只是想存储一个文件，为什么要操作流呢，又不更改文件的内容。

​	所以说，有的时候遇到难以解决的问题就站在一个高度去重新审视这个问题，不要总是急于在网上google答案，一叶障目不见泰山，换个视角换个思维方式。
