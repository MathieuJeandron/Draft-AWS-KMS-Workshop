# Encryption with AWS KMS

Encryption is an integral part of the AWS KMS operations and its interactions with other AWS services. In this section we are going to get a better understanding of it and make some hands-on practices. AWS KMS mostly uses envelope encryption, but can also encrypt data without it. Envelope encryption is the main encryption vehicle for AWS services using AWS KMS. 

The section is divided in the following parts:

* [How Envelope Encryption works in practice](https://github.com/DanGOTO100/Draft-AWS-KMS-Workshop/blob/master/Envolpe-Encryption-in-AWS-KMS.md#how-envelope-encryption-works-in-practice)
* [Envelope encryption. Server Side Encryption](https://github.com/DanGOTO100/Draft-AWS-KMS-Workshop/blob/master/Envolpe-Encryption-in-AWS-KMS.md#envelope-encryption-server-side-encryption)
* [Envelope encryption. Client Side Encryption](https://github.com/DanGOTO100/Draft-AWS-KMS-Workshop/blob/master/Envolpe-Encryption-in-AWS-KMS.md#envelope-encryption-client-side-encryption)
* [Direct Encryption with AWS KMS](https://github.com/DanGOTO100/Draft-AWS-KMS-Workshop/blob/master/Envolpe-Encryption-in-AWS-KMS.md#encryption-using-aws-kms-with-no-data-key)

---

## How Envelope Encryption works in practice

AWS KMS is able to encrypt and decrypt up to 4 kilobytes (4096 bytes) of data. With other volumes of data, normally you will use a data key to perform encryption operations outside KMS through envelope encryption.
Envelope encryption refers to the practice of protecting the data by encrypting it with a data key, and encrypting the data key itself with a another encryption key, a CMK under KMS in this case.
See the following figure from [AWS KMS documentation](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#enveloping).

![alt text](/res/S2F1.png)

<**Figure-1**>


AWS KMS, if you need to, is also capable of generating **data keys** to encrypt data.
**Note:** Data keys are only generated by AWS KMS, not stored or used to encrypt by AWS KMS itself. 
Also, they can be obtained in plain text or encrypted. 

Data keys are very different from a CMK, which never leaves AWS KMS and is only used in memory.
Once the data is encrypted and stored with a data key, the data key is stored encrypted withthe data while the plaintext version is deleted (for security best practices).

Therefore, to decrypt the data, the CMK needs to decrypt the encrypted data key and then, with the data key in plain text,  perform the decrypt operation.
This adds a tier of protection to your data and better manageability. If a data key is compromised, only the service or applicationsusing that particular data key are compromised, instead of the whole platfrom, if we used one key for everything. 

Let's see envelope encryption in action with AWS KMS.



### Step 1 - Create the secret text 

First thing we will do is to create a file with the data we want to encrypt under envelop encryption. A sample "secret" text file in this case.
```
$ sudo echo "Sample Secret Text to Encrypt" > samplesecret.txt
```


### Step 2- Generate the data key

Next, we ask AWS KMS to generate a data key referencing a CMK. The CMK is referenced to encrypt the data key.  We will use the CMK we imported. Use the following command to generate a symmetric data key with 256 bits length.
```
$ aws kms generate-data-key --key-id alias/ImportedCMK --key-spec AES_256 --encryption-context project=workshop
```
You will notice that the command will fail to run. Our current Role with Power User permissions rdoes not have enough privileges to generate a data key. As per least privilege best practices, we are providing permissions as needed in policies attached to the role that can be easily tracked and dettached when such permission for that user is not needed any more.

We need to provide with permission to generate a data key. The process is the same as we used in the previos section of the workshop, please go back to it if you need to in the following link: "[Create a policy to the Role](https://github.com/DanGOTO100/Draft-AWS-KMS-Workshop/blob/master/Working-with-CMKs.md#step-2---create-policy-and-attach-it-to-the-role)"
Basically, you need to go back to the AWS console, in the services area navigate to IAM and go to "**Policies**". We are going to create a new policy and attach it to the Power user role.
As we did in the previous section, click on new "**Create Policy**", Select KMS as the service, go to the Actions area.
In the "Write" section, select "**GenerateDataKey**" and "**Any**" as resource. Click on "**Review Policy**" and then give the policy a name, for example "**KMS-Workshop-GenerateDataKeyPermissions**".

You can click now in "**Create Policy**". Once created, attach it to the role **KMSWorkshop-InstaceInitRole**.

If you try the same command again, it will succeed now. The command will return a JSON with the plaintext key - Plaintext key in b64 encoding, the KeyId used to encrypt it and a CiphetextBlob which is the encrypted data key generated, in base64 enconding. Keep these values, we are going to needed them shortly.

```
$ aws kms generate-data-key --key-id alias/ImportedCMK --key-spec AES_256 --encryption-context project=workshop

{
    "Plaintext": "97**********FEFE*ROL****************OOL", 
    "KeyId": "arn:aws:kms:eu-west-1:-your-account.id:key/-KeyId-used-to-encrypt", 
    "CiphertextBlob": "*********************+**********M05/D********************************************D12bL*****2A**"
}
```

Please note the command used a parameter "**encryption-context**", this is a key-pair optional value that provides additional context about the data and that needs to be provided as well when decrypting the data. This is because, when provided, it is bound to the criptographic operation.

Encruyption Context, besides helping with the integrity of the encrypted data, it has many uses like for example: use to set policies, to provide Grants or to monitor encrypt/decrypt operations in CloudTrail (will see later in this section).
See **additional information about encryption context** in [this part of the AWS KMS  documentation](https://docs.aws.amazon.com/kms/latest/developerguide/encryption-context.html).



### Step 3 - Encrypt the secret text

We have now a data key 256 bit length that we can use to encrypt. We can use any encryption tool or library with the data key we have created to encrypt data. Remember that AWS KMS does not store or use this data key for encryption. 

Let's use OpenSSL library to encrypt the sample file with the data key. We need to decode the plaintext data key we obtained above (Plaintext), as it is in b64, and store it in a file. Name the file with the decoded plaintext key as **datakeyPlainText.txt**, for example. Use a command like the one below, where the argument for "echo" command is the plaintext key obtained in preivous command when creating data key:

```
$ echo '97**********FEFE*ROL****************OOL' | base64 --decode > datakeyPlainText.txt
```


Do the same for the encrypted plaintext (the CipherTextBlob) we have obtained when creating data key,  and save it in a file name **datakeyEncrypted.txt**:

```
$ echo '*********************+**********M05/D********************************************D12bL*****2A**' | base64 --decode > datakeyEncrypted.txt
```
Now we are ready to encrypt the file with the text we want to keep secret. Remember we stored the text in the file **samplesecret.txt**. For the job, we will use the OpenSSL library again, encrypting with AES256:

```
$ openssl enc -e -aes256 -in samplesecret.txt -out encryptedSecret.txt -k fileb://datakeyPlainText.txt  
```

We have our text now encrypted in the file **datakeyPlainText.txt**. You can do a "more" or "cat" operation over it. You will notice the encryption. The original text is not recognizable.

```
$ more encryptedSecret.txt 

FF******f??FZ?***3
```


Following envelope encryption best practices, you will now store the file encrypted with the encrypted data key and delete the plaintext key, avoiding getting it compromised. 

```
$ rm datakeyPlainText.txt
```



### Step 4 - Decrypt the encrypted secret text

If we want now to reverse the operation and decrypt the secret text stored in **encryptedSecret.txt**, it will be a 2 steps procedure. 
Remember that we don´t have the data key in plaintext to be used for decryption, we have removed it for security best practices. 

We have the data key **encrypted** that was provided to us when calling the AWS KMS generate data key operation and that we stored as b64 decoded in file  **datakeyEncrypted.txt**.

Therefore, the first step is to decrypt the data key with the appropriate Master Key (CMK) that was used to encrypt it. In this case we will use the command aws decrypt (see full reference [in this part of documentation](https://docs.aws.amazon.com/cli/latest/reference/kms/decrypt.html)). This command will tell AWS KMS to revert the encryption made over the data key. 

**Note:** We don´t need to provide the KeyId that was used to encrypt the data key, the encrypted data contains metadata to provide this information to AWS KMS.

```
$ aws kms  decrypt --ciphertext-blob fileb://datakeyEncrypted.txt
```

Again, the command will fail because the Power user permissions do not include the capability to decrypt data. We need to create a policy, as we did in [Step 2] (https://github.com/DanGOTO100/Draft-AWS-KMS-Workshop/blob/master/Envolpe-Encryption-in-AWS-KMS.md#step-2--generate-the-data-key) above.
Create a policy first with permission on KMS to "**Decrypt**" and "**Encrypt**". Name the policy **KMS-Workshop-EncDecPermissions**.
Save and attach the policy to the role we are using: **KMSWorkshop-InstaceInitRole**.

Once you added this new policy with the permissions, the command still fails:

```
$ aws kms  decrypt --ciphertext-blob fileb://datakeyEncrypted.txt

An error occurred (InvalidCiphertextException) when calling the Decrypt operation: 

```

This is because when we generated the data key in plaintext and encrypted it with our CMK in the **step 2**, we used an **encryption context**. 
This encryption context was used in the encryption operation of the plaintext data key, this is: to produce the encrypted data key (the CiphertextBlob). Therefore we need to provide the encryption context to be able to decrypt correctly:

```
$ aws kms  decrypt --encryption-context project=workshop --ciphertext-blob fileb://datakeyEncrypted.txt
{
    "Plaintext": "97**********FEFE*ROL****************OOL", 
    "KeyId": "arn:aws:kms:eu-west-1:your-account-id:key/your-imported-key-id"
}
```

Great! As you can see, the output of the command has the data key in Plaintext. We had removed it and only stored it encrypted, but with AWS KMS and the appropriate CMK, we have it back. 

We need now  to decode it from base64 and use it to decrypt the encrypted secret file, **encryptedSecret.txt**. Again using  the OpenSSL library:  Copy the plaintext key and use it in the following commands.

```
$ echo '97**********FEFE*ROL****************OOL' | base64 --decode > datakeyPlainText.txt

$  openssl enc -d -aes256 -in encryptedSecret.txt -k fileb://datakeyPlainText.txt
Sample Secret Text to Encrypt
```

Good job, We have the secret text decrypted and ready to use by our applications. 

By completing this part of the workshop you now have a better understanding of what envelope ecnryption is, let's now see how it applies for AWS services working with AWS KMS.

----


### Envelope encryption. Server side encryption

In AWS there are main two main procedures to protect your data at rest: Client side encryption and Server side Encryption. 

For example in Amazon S3, You can encrypt your data before uploading it into the Amazon S3 Service (client side encryption) or encrypt once the data is there (server side encryption). More details in this [link to the S3 documentation](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingEncryption.html). A similar approach can be taken in other services like Amazon DynamoDB, see [details here](https://docs.aws.amazon.com/dynamodb-encryption-client/latest/devguide/client-server-side.html).

The envelop encryption mechanism described here is also the mechanism behind the server side Encryption that takes place in the AWS services integrated with KMS. There are very good descriptions on how AWS KMS is being used by different services in this part of the [AWS KMS documentation here](https://docs.aws.amazon.com/kms/latest/developerguide/service-integration.html).

For the workshop, let's see an example of attaching a disk to our working instance and how we can encrypt it with AWS KMS and the key we have created with our key material, its alias is "**ImportedCMK**".

We start by going into the AWS console. Navigate to Amazon EC2 service. Look in the left pane. Locate "Elastic Block Store" and click on volumes.


![alt text](/res/S2F2.png)

<**Figure-2**>

Now in the upper area, you can click on "Create Volume" to create a new Amazon EBS disk. Once in the volume creation screen, and for the workshorp you can leave the defaults in most fields- Except for the following: Ensure you have selected the Availability zone finishing in "1a". The disk can only be attached to instances in the same availability zone.
Ensure you have marked  "**Encryption**" checkbox. Then select the CMK you want to use from AWS KMS. Let's select the CMK that we have imported with our own key material, the alias was: **ImportedCMK**.

![alt text](/res/S2F3.png)

<**Figure-3**>

Create a tag for the volume, for example a key-pair like: **Name-WorkshopEB**S.
Click on "**Create volume**" at the right bottom of the screen. The volume will start being created and will be ready to be attached to the instance in a few seconds. 

We have now a encrypted volume that can be attached to our instance. In order to do so, navigating to EC2 service in the console again,  just go to the "**Volumes**" in the left pane. Check Under "**Elastic Block Store**" on the left are of the screen and select the volume you have created. Cick on the "**Actions**" button and select "**Attach**". In the new window, select the instance we want it to be attached to, this is our workshop working instance: **KMSWorkshopInstance**.


If you go back to the terminal connection to your instance, you can see the encrypted 100GiB disk is available for being mounted and used.

```
$ lsblk

NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0    8G  0 disk 
└─xvda1 202:1    0    8G  0 part /
xvdf    202:80   0  100G  0 disk 
```

Behind the scenes, an **envelope encryption process** through AWS KMS has taken place. 

Furthermore, encryption context was used during the process to enforce the security and better traceability.
Amazon EBS request AWS KMS to generate a data key under a CMK. The encrypted data key is stored within the volume metadata and Amazon EC2 ask AWS KMS to decrypt the data key. Then, the content is encrypted with the data key. 

The encryption context is used with the volume-id of the Amazon EBS volume in all the encryption and decryption operations. 
See more details in the following [link to the Amazon EBS documentation](https://docs.aws.amazon.com/kms/latest/developerguide/services-ebs.html).



----

### Envelope encryption. Client side encryption


In the first topic of this section of the workshop (How Envelope Encryption works in practice) we  have just seen an example of how AWS KMS can encrypt in client side by generating a data key, using an envelope encryption mechanism, to encrypt a secret text file with two tiers of protection.

While you can code all the primitives and commands to implement encryption on the client side, AWS has a SDK, the [AWS Encryption SDK](https://docs.aws.amazon.com/es_es/encryption-sdk/latest/developer-guide/introduction.html). This SDK uses envelope encryption and it is integrated with AWS KMS. It will implement most of the operations we have seen before as simple API calls. AWS KMS can become a Master Key provider for the AWS Encryption SDK, however you may use other Master Key provider with the SDK. 

I recommend to take a look at the AWS Encryption SDK. Also, some AWS services have specific encryption SDKs, like the on in Amazon DynamoDB: the "DynamoDB Encryption Client", you can take a look [in this link](https://docs.aws.amazon.com/dynamodb-encryption-client/latest/devguide/what-is-ddb-encrypt.html). 

----


### Encryption using AWS KMS with no Data Key

You can also use the a CMK in AWS KMS to encrypt and decrypt a secret directly, without the generation of a Data Key and hence, without the envelope encryption process. Remember, AWS KMS is able to encrypt and decrypt up to 4 kilobytes (4096 bytes) of data. We can use the CLI or APIs.

 In this example, we will use the CLI to encrypt a secret and decrypt using the CMK itself. Note that the encryption operation takes place in AWS KMS. When a data key is generated via the CMK, the encryption and decryption process happens outside AWS KMS.

Let's Create a new secret file.

```
$ echo "New secret text" > NewSecretFile.txt
```

We are going to encrypt it with a CMK and use an encryption context that will need to be provided in the decrypt operation.

```
$ aws kms encrypt --key-id alias/ImportedCMK --plaintext fileb://NewSecretFile.txt --encryption-context project=kmsworkshop --output text  --query CiphertextBlob | base64 --decode > NewSecretsEncryptedFile.txt
```
Note we have called the CMK used to encrypt by its alias. The input file needs the "fileb" prefix to be processed in binary, while the output is decoded from b64 the new output file "**NewSecretsEncryptedFile.txt**"

Take a look at the Encrypted file:

```
$ cat NewSecretsEncryptedFile.txt
_0]0!!$$?`?He.0?k<n㷇?@PP99Rn0l *?H???@?i$?@ ??+?7??U??b?)?U??;?o?B?P?
```

Great, It unreadable as is encrypted. Can´t read the original text. Now let's decrypt it.

```
$ aws kms decrypt --ciphertext-blob fileb://NewSecretsEncryptedFile.txt --output text --query Plaintext | base64 --decode > NewSecretsDecryptedFile.txt

An error occurred (InvalidCiphertextException) when calling the Decrypt operation:
```

Wait, what is the problem?. At this point of the workshop, we should know that without the encryption context, the operation will not succeed. That was the problem. Let's try it again including the encryption context we stated during the encryption process:

```
$ aws kms decrypt --ciphertext-blob fileb://NewSecretsEncryptedFile.txt --encryption-context project=kmsworkshop --output text --query Plaintext | base64 --decode > NewSecretsDecryptedFile.txt
````

If you now take a look at the file we have created with the previous command "**NewSecretsDecryptedFile.txt**". The secret text is now back unencrypted and ready for us.

You have completed the second section of the workshop. In the next section we will work with a real-life Web App and will try to implement best practices.



