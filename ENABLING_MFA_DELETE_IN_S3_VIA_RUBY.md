***Protect S3 Data from Deletion using Multi-Factor Authentication***
-----
One of the interesting features available in Amazon S3 is Multi Factor Authentication Delete.

This features allows you to specify a TOTP-compatible virtual MFA device or a physical MFA device and link it to a versioned S3 bucket. This means that any future attempt to change the versioning state of the bucket, or delete any version of a file from a bucket MUST be accompanied with a valid token code. (You can’t delete the entire bucket object either – as you can’t delete a bucket holding any files/versions.)

This is especially useful in environments where you want to store security-related audit/event/log data – especially where some level of dual-control and/or physical segregation is required.

When I was trying to see what was available to do this, I looked at a couple of GUI solutions which supported it – however I wanted a scriptable version that could be used in automation scripts. In my case, it was designed to be manually executed by two custodians (one holding the AWS root account credentials, and one holding the physical MFA token). Theoretically, this could be fully automated by having a virtual TOTP application or HSM.

The process to set-up prerequisites is simple:

Install Ruby – I tend to ensure it can run Ruby scripts (.rb files) and is available in PATH.
Install the AWS SDK for Ruby by typing gem install aws-sdk in a command line.
Save the script provided below as a Ruby (.rb) script file
Execute the script by following the usage instructions described below.


**Usage**
```
Structure: 
> enable-mfa-delete-s3.rb ACCESS_KEY SECRET_KEY S3_BUCKET_NAME AWS_ACCOUNT_ID AWS_REGION AWS_TOKEN_CODE

Example: 
> enable-mfa-delete-s3.rb AAAAAAAA BBBBBBBB my-first-s3-bucket 123456789 ap-southeast-2 123456
```


**Script**

```

require 'rubygems'
require 'aws-sdk'

# Fix SSL problem on Windows + Ruby: https://github.com/aws/aws-sdk-core-ruby/issues/166
Aws.use_bundled_cert!

# Read script arguments.
aws_access_key 		= ARGV[0].to_s
aws_secret_key 		= ARGV[1].to_s
aws_bucket_name		= ARGV[2].to_s
aws_account_id 		= ARGV[3].to_s
aws_region 		= ARGV[4].to_s
aws_token_code 		= ARGV[5].to_s

# Create a client object to interface with S3.
client = Aws::S3::Client.new(region: aws_region, :access_key_id => aws_access_key, :secret_access_key => aws_secret_key)

# Assemble the MFA string.
aws_mfa_device_string = 'arn:aws:iam::' + aws_account_id + ':mfa/root-account-mfa-device ' + aws_token_code

# Update the bucket versioning policy.
client.put_bucket_versioning({
  bucket: aws_bucket_name, # required
  mfa: aws_mfa_device_string,
  versioning_configuration: { # required
    mfa_delete: "Enabled", # accepts Enabled, Disabled
    status: "Enabled", # accepts Enabled, Suspended
  },
  use_accelerate_endpoint: false,
})

# Output the new state.
resp = client.get_bucket_versioning({
  bucket: aws_bucket_name, # required
  use_accelerate_endpoint: false,
})

print 'Bucket Name: ' + aws_bucket_name + "\n"
print 'Versioning Status: ' + resp.status + "\n"
print 'MFA Delete Status: ' + resp.mfa_delete + "\n"

```
