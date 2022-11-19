---
title: "terraformでS3バケットへの特定のIPアドレスのアクセスの制限を行う"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws"]
published: false
---

AWS のドキュメントにコードがあるのでそれを terraform に落とし込む
https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/example-bucket-policies.html?utm_source=pocket_reader#example-bucket-policies-use-case-3

コードは GitHub に公開しています。
https://github.com/teitei-tk/terraform-s3-restrict-ip-access-example

# コード

IP アドレスは環境変数経由で渡す。

https://github.com/teitei-tk/terraform-s3-restrict-ip-access-example/blob/main/s3.tf

```terraform
resource "aws_s3_bucket" "bucket" {
  bucket = "terraform-s3-ip-access-example"
  force_destroy = true

  tags = {
    "Name" = "terraform-s3-ip-access-example"
  }
}

resource "aws_s3_bucket_public_access_block" "bucket" {
  bucket = aws_s3_bucket.bucket.id

  block_public_acls = true
  block_public_policy = true
  ignore_public_acls = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_policy" "bucket" {
  bucket =  aws_s3_bucket.bucket.id

  policy = jsonencode(
    {
      "Version": "2008-10-17",
      "Id": "PolicyForBucketIpAccess",
      "Statement": [
        {
          "Sid": "IPAllow",
          "Effect": "Deny",
          "Principal": "*",
          "Action": "s3:*",
          "Resource": [
            "${aws_s3_bucket.bucket.arn}",
            "${aws_s3_bucket.bucket.arn}/*"
          ],
          "Condition": {
            "NotIpAddress": {
              "aws:SourceIp": var.allow_ip_address_block
            }
          }
        }
      ]
    }
```

自分の IP アドレスについては以下のサイトで調べることができる。

https://www.cman.jp/network/support/go_access.cgi

`$ TF_VAR_allow_ip_address_block="192.0.2.1/24" terraform plan`
`$ TF_VAR_allow_ip_address_block="192.0.2.1/24" terraform apply`

## 注意点

考えたら当たり前だが Effect を Allow にすると terraform からアクセスできず、apply 時にエラーが出る

```diff
diff --git a/s3.tf b/s3.tf
index 55279c7..ec98e0b 100644
--- a/s3.tf
+++ b/s3.tf
@@ -26,7 +26,7 @@ resource "aws_s3_bucket_policy" "bucket" {
       "Statement": [
         {
           "Sid": "IPAllow",
-          "Effect": "Deny",
+          "Effect": "Allow",
           "Principal": "*",
           "Action": "s3:*",
           "Resource": [
```

```sh
aws_s3_bucket.bucket: Creating...
aws_s3_bucket.bucket: Creation complete after 3s [id=terraform-s3-ip-access-example]
aws_s3_bucket_public_access_block.bucket: Creating...
aws_s3_bucket_policy.bucket: Creating...
aws_s3_bucket_public_access_block.bucket: Creation complete after 0s [id=terraform-s3-ip-access-example]
╷
│ Error: Error putting S3 policy: AccessDenied: Access Denied
│       status code: 403, request id: hogehoge, host id: fugafuga
│
│   with aws_s3_bucket_policy.bucket,
│   on s3.tf line 19, in resource "aws_s3_bucket_policy" "bucket":
│   19: resource "aws_s3_bucket_policy" "bucket" {
│
╵
```

## 資料

https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/example-bucket-policies.html
https://github.com/teitei-tk/terraform-s3-restrict-ip-access-example
