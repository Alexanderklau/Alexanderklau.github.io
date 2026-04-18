---
categories: ["技术", "Golang"]
title: Golang 调用 aws-sdk 操作 S3对象存储
date: 2019-10-25 09:55:52
tags: ["前后端"]
---
# Golang 调用 aws-sdk 操作 S3对象存储
## 前言
因为业务问题，要写一个S3对象存储管理代码，由于一直写Go，所以这次采用了Go，Go嘛，快，自带多线程，这种好处就不用多说了吧。
## 基础的功能
1. 查看S3中包含的bucket
2. bucket中的文件/文件夹
3. bucket的删除
4. bucket的创建
5. bucket的文件上传
6. bucket的文件下载
7. bucket的文件删除  

## aws-sdk 的安装
玩Golang你还能不会那啥？对吧，那啥？那飞机！那飞机场，安上~  

```bash
go get github.com/aws/aws-sdk-go
```
## aws-sdk-go 的基础使用
### 构建基础的S3连接
访问S3的时候，咱们需要access_key，secret_key，对象存储访问IP这三个参数，我们首先要创建一个aws的config，说白了，我们需要定义aws的配置，这样它才知道要怎么访问，去哪里访问等问题。  
构建一个S3连接代码如下
```golang
package main
import (
	"fmt"
	"os"
    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/credentials"
    _ "github.com/aws/aws-sdk-go/service/s3/s3manager"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3"
)
func main() {

    access_key := "xxxxxxxxxxxxx"
    secret_key := "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
	end_point := "http://xx.xx.xx.xx:7480" //endpoint设置，不要动
	
	sess, err := session.NewSession(&aws.Config{
        Credentials:      credentials.NewStaticCredentials(access_key, secret_key, ""),
        Endpoint:         aws.String(end_point),
        Region:           aws.String("us-east-1"),
        DisableSSL:       aws.Bool(true),
        S3ForcePathStyle: aws.Bool(false), //virtual-host style方式，不要修改
	})
}
```

这时候需要你自己去定义一下access_key，secret_key，end_point这三个参数  
接下来所有的代码，都是以这个连接模板，为核心，后面我就用**同上**代替配置，请注意！  
所有的代码都传到GIT上了，到时候会给出地址，不懂得copy下来吧！
### 查看S3中包含的bucket
查看所有的bucket
```golang
package main
import (
    导入包同上
)

func exitErrorf(msg string, args ...interface{}) {
    fmt.Fprintf(os.Stderr, msg+"\n", args...)
    os.Exit(1)
}

func main() {

    配置同上

	svc := s3.New(sess)
	result, err := svc.ListBuckets(nil)
	if err != nil {
		exitErrorf("Unable to list buckets, %v", err)
	}
	
	fmt.Println("Buckets:")
	
	for _, b := range result.Buckets {
		fmt.Printf("* %s created on %s\n",
			aws.StringValue(b.Name), aws.TimeValue(b.CreationDate))
	}

	for _, b := range result.Buckets {
		fmt.Printf("%s\n", aws.StringValue(b.Name))
	}
	
}
```
### 列出bucket中的文件/文件夹
查看某个bucket中包含的文件/文件夹
```golang
package main
import (
    "github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/aws/credentials"
    "github.com/aws/aws-sdk-go/service/s3"
    "fmt"
    "os"
)

func exitErrorf(msg string, args ...interface{}) {
    fmt.Fprintf(os.Stderr, msg+"\n", args...)
    os.Exit(1)
}

func main() {

    配置同上

    // bucket后跟，go run ....go bucketname
    bucket := os.Args[1]
	fmt.Printf(bucket)
	fmt.Printf("\n")

	svc := s3.New(sess)

	params := &s3.ListObjectsInput{
		Bucket:             aws.String(bucket), 
    }
	resp, err := svc.ListObjects(params)

	if err != nil {
		exitErrorf("Unable to list items in bucket %q, %v", bucket, err)
	}
	
	for _, item := range resp.Contents {
		fmt.Println("Name:         ", *item.Key)
		fmt.Println("Last modified:", *item.LastModified)
		fmt.Println("Size:         ", *item.Size)
		fmt.Println("Storage class:", *item.StorageClass)
		fmt.Println("")
	}
	
}
```
### bucket的创建
创建bucket
```golang
package main

import (
    导包同上
)


func exitErrorf(msg string, args ...interface{}) {
    fmt.Fprintf(os.Stderr, msg+"\n", args...)
    os.Exit(1)
}

func main() {

    配置同上

	bucket := os.Args[1]

	if len(os.Args) != 2 {
		exitErrorf("Bucket name required\nUsage: %s bucket_name",
			os.Args[0])
	}
	

	// Create S3 service client
	svc := s3.New(sess)

	params := &s3.CreateBucketInput{
		Bucket: aws.String(bucket),
 }

	_, err = svc.CreateBucket(params)

	if err != nil {
		exitErrorf("Unable to create bucket %q, %v", bucket, err)
	}
	
	// Wait until bucket is created before finishing
	fmt.Printf("Waiting for bucket %q to be created...\n", bucket)
	
	err = svc.WaitUntilBucketExists(&s3.HeadBucketInput{
		Bucket: aws.String(bucket),
	})

	if err != nil {
		exitErrorf("Error occurred while waiting for bucket to be created, %v", bucket)
	}
	
	fmt.Printf("Bucket %q successfully created\n", bucket)
}
```
### bucket的文件上传
往某个固定的bucket里传文件
```golang
package main

import (
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/service/s3/s3manager"
    "fmt"
    "os"
)


func exitErrorf(msg string, args ...interface{}) {
    fmt.Fprintf(os.Stderr, msg+"\n", args...)
    os.Exit(1)
}

func main() {

    配置同上

	if len(os.Args) != 3 {
		exitErrorf("bucket and file name required\nUsage: %s bucket_name filename",
			os.Args[0])
	}
	
	bucket := os.Args[1]
	filename := os.Args[2]
	
	file, err := os.Open(filename)
	if err != nil {
		exitErrorf("Unable to open file %q, %v", err)
	}
	
	defer file.Close()

	uploader := s3manager.NewUploader(sess)

	_, err = uploader.Upload(&s3manager.UploadInput{
		Bucket: aws.String(bucket),
		Key: aws.String(filename),
		Body: file,
	})
	if err != nil {
		// Print the error and exit.
		exitErrorf("Unable to upload %q to %q, %v", filename, bucket, err)
	}
	
	fmt.Printf("Successfully uploaded %q to %q\n", filename, bucket)
}
```
### bucket的文件下载
下载某个bucket中的某个文件
```golang
package main

import (
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/service/s3"
	"github.com/aws/aws-sdk-go/service/s3/s3manager"

    "fmt"
    "os"
)


func exitErrorf(msg string, args ...interface{}) {
    fmt.Fprintf(os.Stderr, msg+"\n", args...)
    os.Exit(1)
}

func main() {

    配置同上

	if len(os.Args) != 3 {
		exitErrorf("Bucket and item names required\nUsage: %s bucket_name item_name",
			os.Args[0])
	}
	
	bucket := os.Args[1]
	item := os.Args[2]

	file, err := os.Create(item)
    if err != nil {
        exitErrorf("Unable to open file %q, %v", err)
    }

    defer file.Close()
	
	downloader := s3manager.NewDownloader(sess)

	numBytes, err := downloader.Download(file,
    &s3.GetObjectInput{
        Bucket: aws.String(bucket),
        Key:    aws.String(item),
    })
if err != nil {
    exitErrorf("Unable to download item %q, %v", item, err)
}

fmt.Println("Downloaded", file.Name(), numBytes, "bytes")
}	
```
### bucket的文件删除
删除某个bucket里面的某个文件
```golang
package main

import (
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/service/s3"
    "fmt"
    "os"
)


func exitErrorf(msg string, args ...interface{}) {
    fmt.Fprintf(os.Stderr, msg+"\n", args...)
    os.Exit(1)
}

func main() {

    配置同上

	if len(os.Args) != 3 {
		exitErrorf("Bucket and object name required\nUsage: %s bucket_name object_name",
			os.Args[0])
	}
	
	bucket := os.Args[1]
	obj := os.Args[2]
	
	svc := s3.New(sess)

	_, err = svc.DeleteObject(&s3.DeleteObjectInput{Bucket: aws.String(bucket), Key: aws.String(obj)})
	if err != nil {
		exitErrorf("Unable to delete object %q from bucket %q, %v", obj, bucket, err)
	}

	err = svc.WaitUntilObjectNotExists(&s3.HeadObjectInput{
		Bucket: aws.String(bucket),
		Key:    aws.String(obj),
	})

	fmt.Printf("Object %q successfully deleted\n", obj)
}
```
## 代码所在地
https://github.com/Alexanderklau/Go_poject/tree/master/Go-Storage
