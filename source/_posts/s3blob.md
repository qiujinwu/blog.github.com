---
title: Golang对象存储接口
date: 2020-01-14 00:00:00
tags:
 - golang
categories:
 - Golang
---

``` golang
package main

import (
	"context"
	"fmt"
	"io"
	"os"
	"path"
	"path/filepath"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/aws/session"
	"gocloud.dev/blob"
	"gocloud.dev/blob/fileblob"
	"gocloud.dev/blob/s3blob"
)

const (
	bucketName = "kimq"
)

type blobStorage struct {
	driver string
	bucket *blob.Bucket
}

func (u *blobStorage) Upload(ctx context.Context, name string, reader io.Reader) (outputURL string, err error) {
	key := path.Join("images/", name)
	writer, err := u.bucket.NewWriter(ctx, key, nil)
	if err != nil {
		return "", fmt.Errorf("new writer error: %v", err)
	}

	_, err = io.Copy(writer, reader)
	if err != nil {
		return "", fmt.Errorf("io copy error: %v", err)
	}

	defer writer.Close()

	if err := writer.Close(); err != nil {
		return "", fmt.Errorf("writer close error: %v", err)
	}

	ourl := fmt.Sprintf("%s://%s/%s", u.driver, bucketName, key)
	return ourl, nil
}

func init_file_bucket(ctx context.Context, bucket string) (*blobStorage, error) {
	dir := filepath.Join("/tmp/1/", bucket)
	//dir := filepath.Join(os.TempDir(), bucket)
	if err := os.MkdirAll(dir, 0775); err != nil {
		return nil, fmt.Errorf("create bucket dir %q error: %v", dir, err)
	}

	bb, err := fileblob.OpenBucket(dir, nil)
	if err != nil {
		return nil, fmt.Errorf("open bucket error: %v", err)
	}

	return &blobStorage{
		driver: "file",
		bucket: bb,
	}, nil
}

func init_jd_bucket(ctx context.Context, bucket string) (*blobStorage, error) {
	ak := "Access Key ID"
	sk := "Access Key Secret"

	sess, err := session.NewSession(&aws.Config{
		Endpoint:         aws.String("s3.cn-south-1.jdcloud-oss.com"), //Bucket所在Endpoint
		Region:           aws.String("cn-south-1"),                    //Bucket所在Region
		S3ForcePathStyle: aws.Bool(true),
		Credentials:      credentials.NewStaticCredentials(ak, sk, ""),
	})

	if err != nil {
		return nil, fmt.Errorf("new session error: %v", err)
	}

	sb, err := s3blob.OpenBucket(ctx, sess, bucket, nil)
	if err != nil {
		return nil, fmt.Errorf("open bucket error: %v", err)
	}

	return &blobStorage{
		driver: "jds3",
		bucket: sb,
	}, nil

}

func main() {
	path := "/tmp/workshop.png"
	filename := filepath.Base(path)
	f, err := os.Open(path)
	if err != nil {
		panic(err)
	}

	ctx := context.Background()

	// up, err := init_file_bucket(ctx, bucketName)
	up, err := init_jd_bucket(ctx, bucketName)
	if err != nil {
		panic(err)
	}

	if f, err := up.Upload(ctx, filename, f); err == nil {
		fmt.Printf("%s uploader ok %s\n", up.driver, f)
	} else {
		panic(err)
	}
}
```

# 参考
+ <https://www.alibabacloud.com/help/zh/doc-detail/64919.htm>
+ <https://docs.jdcloud.com/cn/object-storage-service/introduction-2>
+ <https://docs.jdcloud.com/cn/object-storage-service/regions-and-endpoints>
