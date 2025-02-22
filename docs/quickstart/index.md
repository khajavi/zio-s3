---
id: quickstart_index
title: "Quick Start"
---

Setup
-----

```
//support scala 2.12 / 2.13

libraryDependencies += "dev.zio" %% "zio-s3" % "<version>"
```

How to use it ?
---------------

ZIO-S3 is a thin wrapper over the s3 async java client. It exposes the main operations of the s3 java client.


```scala
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials
import zio.Chunk
import zio.s3._
import zio.stream.{ZSink, ZStream}
import software.amazon.awssdk.services.s3.model.S3Exception

  // list all buckets available  
  listBuckets.provideLayer(
     live("us-east-1", AwsBasicCredentials.create("accessKeyId", "secretAccessKey"))
  )
  
  // list all objects of all buckets
  val l2: ZStream[S3, S3Exception, String] = (for {
     bucket <- ZStream.fromIterableZIO(listBuckets) 
     obj <- listAllObjects(bucket.name)
  } yield obj.bucketName + "/" + obj.key).provideLayer(
     live("us-east-1", AwsBasicCredentials.create("accessKeyId", "secretAccessKey"))
  )  
```

All available s3 combinators and operations are available in the package object `zio.s3`, you only need to `import zio.s3._`


Credentials
-----------

zio-s3 expose credentials providers from aws https://docs.aws.amazon.com/sdk-for-java/v2/developer-guide/credentials.html
If credentials cannot be found in one or multiple providers selected the operation will fail with `InvalidCredentials`

```scala
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials
import zio._
import software.amazon.awssdk.regions.Region
import software.amazon.awssdk.services.s3.model.S3Exception
import zio.s3._
import zio.s3.providers._

// build S3 Layer from basic credentials
val s3: Layer[S3Exception, S3] =
  live(Region.AF_SOUTH_1, AwsBasicCredentials.create("key", "secret"))

// build S3 Layer from System properties or Environment variables
val s3: Layer[S3Exception, S3] =
  liveZIO(Region.AF_SOUTH_1, system <> env)

// build S3 Layer  from Instance profile credentials
val s3: Layer[S3Exception, S3] =
  liveZIO(Region.AF_SOUTH_1, instanceProfile)

// build S3 Layer from web identity token credentials with STS. awssdk sts module required to be on classpath
val s3: Layer[S3Exception, S3] = liveZIO(Region.AF_SOUTH_1, webIdentity)

// build S3 Layer from default available credentials providers
val s3: Layer[S3Exception, S3] = liveZIO(Region.AF_SOUTH_1, default)

// use custom logic to fetch aws credentials
val zcredentials: ZIO[R, S3Exception, AwsCredentials] = ??? // specific implementation to fetch credentials
val s3: ZLayer[Any, S3Exception, S3] = settings(Region.AF_SOUTH_1, zcredentials) >>> live


```

Test / Stub
-----------

a stub implementation of s3 storage is provided for testing purpose and use internally a filesystem to simulate s3 storage

```scala
import zio.nio.core.file.{Path => ZPath}
import zio.s3._

// build s3 Layer
val stubS3: ZLayer[Any, Nothing, S3] = stub(ZPath("/tmp/s3-data"))

// list all buckets available by using S3 Stub Layer 
// will list all directories of `/tmp/s3-data`
listBuckets.provideLayer(stubS3) 
```

More information here on how to use [ZLayer https://zio.dev/docs/howto/howto_use_layers](https://zio.dev/docs/howto/howto_use_layers)


Examples
--------

```scala
import software.amazon.awssdk.services.s3.model.S3Exception
import zio._
import zio.stream.{ ZSink, ZStream }
import zio.s3._

// upload
val json: Chunk[Byte] = Chunk.fromArray("""{  "id" : 1 , "name" : "A1" }""".getBytes)
val up: ZIO[S3, S3Exception, Unit] = putObject(
  "bucket-1",
  "user.json",
  json.length,
  ZStream.fromChunk(json),
  UploadOptions.fromContentType("application/json")
)

// multipartUpload 
import java.io.FileInputStream
import java.nio.file.Paths

val is = ZStream.fromInputStream(new FileInputStream(Paths.get("/my/path/to/myfile.zip").toFile))
val proc2: ZIO[S3, S3Exception, Unit] =
  multipartUpload(
    "bucket-1",
    "upload/myfile.zip",
    is,
    MultipartUploadOptions.fromUploadOptions(UploadOptions.fromContentType("application/zip"))
  )(4)

// download
import java.io.OutputStream

val os: OutputStream = ???
val proc3: ZIO[S3, Exception, Long] = getObject("bucket-1", "upload/myfile.zip").run(ZSink.fromOutputStream(os))
```

Support any commands ?
---

If you need a method which is not wrapped by the library, you can have access to underlying S3 client in a safe manner by using

```scala
import java.util.concurrent.CompletableFuture
import zio.s3._
import software.amazon.awssdk.services.s3.S3AsyncClient
 
def execute[T](f: S3AsyncClient => CompletableFuture[T]) 
```
