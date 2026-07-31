# Module 16: AWS S3 and Boto3 - Moving File Uploads to the Cloud

Currently, we manage images directly in a local folder within our project. This is not suitable for a production environment, as local files are lost during server deployments or scaling. We need to move them to **Object Storage**—a cloud-based architecture designed to store and serve static assets like images, videos, and documents. In this module, we move our file uploads to Amazon S3 (AWS's object storage service).

## 1. Installing Boto3
The first thing we need to do is install `boto3`. This is the official AWS SDK for Python and the standard library used to communicate with AWS services.
```bash
uv add boto3
```

## 2. Creating and Configuring the S3 Bucket
In the AWS Console, create a new S3 bucket with the default configuration, except for one critical change in the **Block Public Access** settings. To allow public viewing of profile pictures, You must uncheck all options except the following two:

* Block public access to buckets and objects granted through new access control lists (ACLs)

* Block public access to buckets and objects granted through any access control lists (ACLs)

## 3. Setting the S3 Bucket Policy (Public Read)
To allow frontend users (browsers) to actually see the profile pictures, we need to attach a Bucket Policy that grants public read access to the specific images folder.

* Paste this into `Amazon S3 > Buckets > <NameOfBucket> > Permissions > Bucket Policy`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadProfilePics",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::fastapi-blog-uploads/profile_pics/*"
    }
  ]
}
```

## 4. Setting the IAM Policy & User (Backend Access)
To allow our FastAPI backend to upload and delete files, we need to create an IAM (Identity and Access Management) policy and attach it to a new backend user.

* **Create the Policy (JSON):**

> **Note:** Be sure to modify `fastapi-blog-uploads` to match your actual bucket name!

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::fastapi-blog-uploads/profile_pics/*"
    }
  ]
}
```

* **Create the User:** Create an IAM user, attach this new policy to them, and generate an Access Key. AWS will provide a Public Key and a Secret Key.

## 5. Environment Variables & Configuration
We must save our AWS credentials securely so the application can authenticate with AWS.

* **Update `.env`:**
```txt
S3_BUCKET_NAME=fastapi-blog-uploads
S3_REGION=us-east-1
S3_ACCESS_KEY_ID=your-access-key-id
S3_SECRET_ACCESS_KEY=your-secret-access-key
```

* **Update `config.py`:**
```python
# S3 Configuration
s3_bucket_name: str
s3_region: str = "us-east-1"
s3_access_key_id: SecretStr | None = None
s3_secret_access_key: SecretStr | None = None
s3_endpoint_url: str | None = None
```

## 6. Updating Image Utilities (`image_utils.py`)
We refactored our utility file to handle cloud logic. We added four new functions:

1. Two functions use `boto3` to upload and delete images on S3.

2. Two "wrapper" functions execute the `boto3` calls using FastAPI's `run_in_threadpool`.
  * Why? Boto3 is a synchronous library. If we run it normally, it will block our async FastAPI server. Wrapping it in a threadpool allows the server to remain asynchronous while the file uploads in the background.

## 7. Updating Database Models (`models.py`)
Because the files no longer live on our local server, the API needs to return the public AWS URL for the image. We updated the model's property to dynamically generate the S3 link:

```python
return f"https://{settings.s3_bucket_name}.s3.{settings.s3_region}[.amazonaws.com/profile_pics/](https://.amazonaws.com/profile_pics/){self.image_file}"
```
## 8. Updating the Routers (`routers/users.py`)
Finally, we updated the upload route to be asynchronous and added error handling specifically for AWS. If S3 goes down or rejects the credentials, it throws a `ClientError`, which we catch and convert into a clean HTTP 500 response.
```python
from botocore.exceptions import ClientError

# Upload to S3 (runs in threadpool via our async wrapper)
try:
    await upload_profile_image(processed_bytes, new_filename)
except ClientError as err:
    raise HTTPException(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        detail="Failed to upload image. Please try again.",
    ) from err
```

## 9. Module Summary  
In this module, we successfully decoupled file storage from our application server. By configuring an AWS S3 bucket with public read access and a restricted IAM user with write access, our FastAPI app can securely upload and delete profile pictures using `boto3`. We also ensured our API remains non-blocking by offloading synchronous AWS network calls to a threadpool, making our application completely ready for scalable production deployment.

# Return to Readme.md
[**Readme.md**](../README.md)