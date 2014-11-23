meteor-slingshot
================

Direct and secure file-uploads to AWS S3, Google Cloud Storage and others.

## Install

```bash
meteor add edgee:slingshot
```

## Why?

There are many many packages out there that allow file uploads to S3,
Google Cloud and other cloud storage services, but they usually rely on the
meteor apps' server to relay the files to the cloud service, which puts the
server under unnecessary load.

meteor-slingshot uploads the files directly to the cloud service from the
browser without ever exposing your secret access key or any other sensitive data
to the client and without requiring public write access to cloud storage to the
entire public.

File uploads can not only be restricted by file-size and file-type, but also by
other stateful criteria such as the current meteor user.

## Quick Example

### Client side

On the client side we can now upload files through to the bucket:

```JavaScript
var uploader = Slingshot.upload("myFileUploads");

uploader.send(document.getElementById('input').files[0], function (error, url) {
  Meteor.users.update(Meteor.userId(), {$push: {"profile.files": url}});
});
```

### Server side

On the server we declare a directive that controls upload access rules:

```JavaScript
Slingshot.createDirective("myFileUploads", Slingshot.S3Storage, {
  bucket: "mybucket",
  allowedFileTypes: ["image/png", "image/jpeg", "image/gif"],

  authorize: function () {
    //Deny uploads if user is not logged in.
    if (!this.userId) {
      var message = "Please login before posting files";
      throw new Meteor.Error("Login Required", message);
    }
  },

  key: function (file) {
    //Store file into a directory by the user's username.
    var user = Meteor.users.findOne(this.userId);
    return user.username + "/" + file.name;
  }
});
```

This directive will not allow any files other than images to be uploaded. The
policy is directed by the meteor app server and enforced by AWS S3.

## Storage services

The client side is no

### AWS S3

You will need a`AWSAccessKeyId` and `AWSSecretAccessToken` ins `Meteor.settings`
and a bucket with the following CORS configuration:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>PUT</AllowedMethod>
        <AllowedMethod>POST</AllowedMethod>
        <AllowedMethod>GET</AllowedMethod>
        <AllowedMethod>HEAD</AllowedMethod>
        <MaxAgeSeconds>3000</MaxAgeSeconds>
        <AllowedHeader>*</AllowedHeader>
    </CORSRule>
</CORSConfiguration>
```

Declare AWS S3 Directives as follows:

```JavaScript
Slingshot.createDirective("aws-s3-example", Slingshot.S3Storage, {
  //...
});
```

### Google Cloud

[Generate a private key](http://goo.gl/kxt5qz) and convert it to a `.pem` file
using openssl:

```
openssl pkcs12 -in google-cloud-service-key.p12 -nodes -nocerts > google-cloud-service-key.pem
```

Save this file into the `/private` directory of your meteor app and add this
line to your server-side code:

```JavaScript
Slingshot.GoogleCloud.defaultDirective.GoogleSecretKey = Assets.getText('google-cloud-service-key.pem');
```
Declare Google Cloud Storage Directives as follows:

```JavaScript
Slingshot.createDirective("google-cloud-example", Slingshot.GoogleCloud, {
  //...
});
```

## Reference

### Directives

`authorize`: Function (required) - Function to determines if upload is allowed.

`maxSize`: Number (required) - Maximum file-size (in bytes). Use null for
unlimited.

`allowedFileTypes` RegExp, String or Array (required) - Allowed MIME types. Use
null for any file type.

`cacheControl` String (optional) - RFC 2616 Cache-Control directive

`contentDisposition` String (required) - RFC 2616 Content-Disposition directive.
Default is the uploaded file's name (inline). Use null to disable.

`bucket` String (required) - Name of bucket to use.

`key` String or Function (required) - Name of the file on the cloud storage
service. If a function is provided, it will be called with `userId` in the
context and its return value is used as the key.

`expire` Number (required) - Number of milliseconds in which an upload
authorization will expire after the request was made.

`acl` String (optional)

`AWSAccessKeyId` String (required for AWS S3)
`AWSSecretAccessKey` String (required for AWS S3)

`GoogleAccessId` String (required for Google Cloud Storage)
`GoogleSecretKey` String (required for Google Cloud Storage)
