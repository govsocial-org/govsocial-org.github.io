---
#template: home.html
title: S3 Storage
---

# S3 Storage

Most Fediverse platforms require some form of S3-compatible storage for images and other media files.

We used [Google Cloud Storage](https://cloud.google.com/storage/docs/discover-object-storage-console) for this. Your bucket will need to be `public`, with `fine-grained` access control, and use the `standard` [storage class](https://cloud.google.com/storage/docs/storage-classes#descriptions).

The service account principal doesn't need any roles - you'll use a [HMAC key](https://cloud.google.com/storage/docs/authentication/managing-hmackeys) for access.