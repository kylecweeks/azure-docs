---
title: How to use blob storage (object storage) from PHP | Microsoft Docs
description: Store unstructured data in the cloud with Azure Blob storage (object storage).
documentationcenter: php
services: storage
author: tamram
manager: timlt
editor: tysonn

ms.assetid: 1af56b59-b3f0-4b46-8441-aab463ae088e
ms.service: storage
ms.workload: storage
ms.tgt_pltfrm: na
ms.devlang: PHP
ms.topic: article
ms.date: 01/11/2018
ms.author: tamram

---
# How to use blob storage from PHP
[!INCLUDE [storage-selector-blob-include](../../../includes/storage-selector-blob-include.md)]

[!INCLUDE [storage-try-azure-tools-queues](../../../includes/storage-try-azure-tools-blobs.md)]

## Overview
Azure Blob storage is a service that stores unstructured data in the cloud as objects/blobs. Blob storage can store any type of text or binary data, such as a document, media file, or application installer. Blob storage is also referred to as object storage.

This guide shows you how to perform common scenarios using the Azure blob service. The samples are written in PHP and use the [Azure Storage Blob Client Library for PHP][download]. The scenarios covered include **uploading**, **listing**, **downloading**, and **deleting** blobs. For more information on blobs, see the [Next steps](#next-steps) section.

[!INCLUDE [storage-blob-concepts-include](../../../includes/storage-blob-concepts-include.md)]

[!INCLUDE [storage-create-account-include](../../../includes/storage-create-account-include.md)]

## Create a PHP application
The only requirement for creating a PHP application that accesses the Azure blob service is the referencing of classes in the [Azure Storage Client Library for PHP][download] from within your code. You can use any development tools to create your application, including Notepad.

In this guide, you use the Blob storage service features, which can be called within a PHP application locally or in code running within an Azure web role, worker role, or website.

## Get the Azure Client Libraries
### Install via Composer
1. Create a file named **composer.json** in the root of your project and add the following code to it:
   
    ```json
    {
      "require": {
        "microsoft/azure-storage-blob": "*"
      }
    }
    ```
2. Download **[composer.phar][composer-phar]** in your project root.
3. Open a command prompt and execute the following command in your project root
   
    ```
    php composer.phar install
    ```

Alternatively go to the [Azure Storage PHP Client Library][download] on GitHub to clone the source code.

## Configure your application to access the blob service
To use the Azure blob service APIs, you need to:

1. Reference the autoloader file using the [require_once] statement, and
2. Reference any classes you might use.

The following example shows how to include the autoloader file and reference the **BlobRestProxy** class.

```php
require_once 'vendor/autoload.php';
use MicrosoftAzure\Storage\Blob\BlobRestProxy;
```

In the examples below, the `require_once` statement is shown always, but only the classes necessary for the example to execute are referenced.

## Set up an Azure storage connection
To instantiate an Azure blob service client, you must first have a valid connection string. The format for the blob service connection string is:

For accessing a live service:

```php
DefaultEndpointsProtocol=[http|https];AccountName=[yourAccount];AccountKey=[yourKey]
```

For accessing the storage emulator:

```php
UseDevelopmentStorage=true
```

To create an Azure Blob service client, you need to use the **BlobRestProxy** class. You can:

* Pass the connection string directly to it or
* Use environment variables in your Web App to store the connection string. See [Azure web app configuration settings](../../app-service/web-sites-configure.md) document for configuring connection strings.

For the examples outlined here, the connection string is passed directly.

```php
require_once 'vendor/autoload.php';

use MicrosoftAzure\Storage\Blob\BlobRestProxy;
use MicrosoftAzure\Storage\Common\Exceptions\ServiceException;

$connectionString = "DefaultEndpointsProtocol=http;AccountName=<accountNameHere>;AccountKey=<accountKeyHere>";

$blobClient = BlobRestProxy::createBlobService($connectionString);
```

## Create a container
[!INCLUDE [storage-container-naming-rules-include](../../../includes/storage-container-naming-rules-include.md)]

A **BlobRestProxy** object lets you create a blob container with the **createContainer** method. When creating a container, you can set options on the container, but doing so is not required. (The following example shows how to set the container access control list (ACL) and container metadata.)

```php
require_once 'vendor\autoload.php';

use MicrosoftAzure\Storage\Blob\BlobRestProxy;
use MicrosoftAzure\Storage\Common\Exceptions\ServiceException;
use MicrosoftAzure\Storage\Blob\Models\CreateContainerOptions;
use MicrosoftAzure\Storage\Blob\Models\PublicAccessType;

$connectionString = "DefaultEndpointsProtocol=http;AccountName=<accountNameHere>;AccountKey=<accountKeyHere>";

// Create blob client.
$blobClient = BlobRestProxy::createBlobService($connectionString);


// OPTIONAL: Set public access policy and metadata.
// Create container options object.
$createContainerOptions = new CreateContainerOptions();

// Set public access policy. Possible values are
// PublicAccessType::CONTAINER_AND_BLOBS and PublicAccessType::BLOBS_ONLY.
// CONTAINER_AND_BLOBS:
// Specifies full public read access for container and blob data.
// proxys can enumerate blobs within the container via anonymous
// request, but cannot enumerate containers within the storage account.
//
// BLOBS_ONLY:
// Specifies public read access for blobs. Blob data within this
// container can be read via anonymous request, but container data is not
// available. proxys cannot enumerate blobs within the container via
// anonymous request.
// If this value is not specified in the request, container data is
// private to the account owner.
$createContainerOptions->setPublicAccess(PublicAccessType::CONTAINER_AND_BLOBS);

// Set container metadata.
$createContainerOptions->addMetaData("key1", "value1");
$createContainerOptions->addMetaData("key2", "value2");

try    {
    // Create container.
    $blobClient->createContainer("mycontainer", $createContainerOptions);
}
catch(ServiceException $e){
    // Handle exception based on error codes and messages.
    // Error codes and messages are here:
    // http://msdn.microsoft.com/library/azure/dd179439.aspx
    $code = $e->getCode();
    $error_message = $e->getMessage();
    echo $code.": ".$error_message."<br />";
}
```

Calling **setPublicAccess(PublicAccessType::CONTAINER\_AND\_BLOBS)** makes the container and blob data accessible via anonymous requests. Calling **setPublicAccess(PublicAccessType::BLOBS_ONLY)** makes only blob data accessible via anonymous requests. For more information about container ACLs, see [Set container ACL (REST API)][container-acl].

For more information about Blob service error codes, see [Blob Service Error Codes][error-codes].

## Upload a blob into a container
To upload a file as a blob, use the **BlobRestProxy->createBlockBlob** method. This operation creates the blob if it doesn't exist, or overwrites it if it does. The following code example assumes that the container has already been created and uses [fopen][fopen] to open the file as a stream.

```php
require_once 'vendor/autoload.php';

use MicrosoftAzure\Storage\Blob\BlobRestProxy;
use MicrosoftAzure\Storage\Common\Exceptions\ServiceException;

$connectionString = "DefaultEndpointsProtocol=http;AccountName=<accountNameHere>;AccountKey=<accountKeyHere>";

// Create blob REST proxy.
$blobClient = BlobRestProxy::createBlobService($connectionString);

$content = fopen("c:\myfile.txt", "r");
$blob_name = "myblob";

try    {
    //Upload blob
    $blobClient->createBlockBlob("mycontainer", $blob_name, $content);
}
catch(ServiceException $e){
    // Handle exception based on error codes and messages.
    // Error codes and messages are here:
    // http://msdn.microsoft.com/library/azure/dd179439.aspx
    $code = $e->getCode();
    $error_message = $e->getMessage();
    echo $code.": ".$error_message."<br />";
}
```

Note that the previous sample uploads a blob as a stream. However, a blob can also be uploaded as a string using, for example, the [file\_get\_contents][file_get_contents] function. To do this using the previous sample, change `$content = fopen("c:\myfile.txt", "r");` to `$content = file_get_contents("c:\myfile.txt");`.

## List the blobs in a container
To list the blobs in a container, use the **BlobRestProxy->listBlobs** method with a **foreach** loop to loop through the result. The following code displays the name of each blob as output in a container and displays its URI to the browser.

```php
require_once 'vendor/autoload.php';

use MicrosoftAzure\Storage\Blob\BlobRestProxy;
use MicrosoftAzure\Storage\Common\Exceptions\ServiceException;

// Create blob REST proxy.
$blobClient = BlobRestProxy::createBlobService($connectionString);

try    {
    // List blobs.
    $blob_list = $blobClient->listBlobs("mycontainer");
    $blobs = $blob_list->getBlobs();

    foreach($blobs as $blob)
    {
        echo $blob->getName().": ".$blob->getUrl()."<br />";
    }
}
catch(ServiceException $e){
    // Handle exception based on error codes and messages.
    // Error codes and messages are here:
    // http://msdn.microsoft.com/library/azure/dd179439.aspx
    $code = $e->getCode();
    $error_message = $e->getMessage();
    echo $code.": ".$error_message."<br />";
}
```

## Download a blob
To download a blob, call the **BlobRestProxy->getBlob** method, then call the **getContentStream** method on the resulting **GetBlobResult** object.

```php
require_once 'vendor/autoload.php';

use MicrosoftAzure\Storage\Blob\BlobRestProxy;
use MicrosoftAzure\Storage\Common\Exceptions\ServiceException;

// Create blob REST proxy.
$blobClient = BlobRestProxy::createBlobService($connectionString);


try    {
    // Get blob.
    $blob = $blobClient->getBlob("mycontainer", "myblob");
    fpassthru($blob->getContentStream());
}
catch(ServiceException $e){
    // Handle exception based on error codes and messages.
    // Error codes and messages are here:
    // http://msdn.microsoft.com/library/azure/dd179439.aspx
    $code = $e->getCode();
    $error_message = $e->getMessage();
    echo $code.": ".$error_message."<br />";
}
```

Note that the example above gets a blob as a stream resource (the default behavior). However, you can use the [stream\_get\_contents][stream-get-contents] function to convert the returned stream to a string.

## Delete a blob
To delete a blob, pass the container name and blob name to **BlobRestProxy->deleteBlob**.

```php
require_once 'vendor/autoload.php';

use MicrosoftAzure\Storage\Blob\BlobRestProxy;
use MicrosoftAzure\Storage\Common\Exceptions\ServiceException;

// Create blob REST proxy.
$blobClient = BlobRestProxy::createBlobService($connectionString);

try    {
    // Delete blob.
    $blobClient->deleteBlob("mycontainer", "myblob");
}
catch(ServiceException $e){
    // Handle exception based on error codes and messages.
    // Error codes and messages are here:
    // http://msdn.microsoft.com/library/azure/dd179439.aspx
    $code = $e->getCode();
    $error_message = $e->getMessage();
    echo $code.": ".$error_message."<br />";
}
```

## Delete a blob container
Finally, to delete a blob container, pass the container name to **BlobRestProxy->deleteContainer**.

```php
require_once 'vendor/autoload.php';

use MicrosoftAzure\Storage\Blob\BlobRestProxy;
use MicrosoftAzure\Storage\Common\Exceptions\ServiceException;

// Create blob REST proxy.
$blobClient = BlobRestProxy::createBlobService($connectionString);

try    {
    // Delete container.
    $blobClient->deleteContainer("mycontainer");
}
catch(ServiceException $e){
    // Handle exception based on error codes and messages.
    // Error codes and messages are here:
    // http://msdn.microsoft.com/library/azure/dd179439.aspx
    $code = $e->getCode();
    $error_message = $e->getMessage();
    echo $code.": ".$error_message."<br />";
}
```

## Next steps
Now that you've learned the basics of the Azure blob service, follow these links to learn about more complex storage tasks.

* Visit the [API Reference for Azure Storage PHP Client Library](http://azure.github.io/azure-storage-php/)
* See the [Advanced Blob example](https://github.com/Azure/azure-storage-php/blob/master/samples/BlobSamples.php).

[download]: https://github.com/Azure/azure-storage-php
[container-acl]: http://msdn.microsoft.com/library/azure/dd179391.aspx
[error-codes]: http://msdn.microsoft.com/library/azure/dd179439.aspx
[file_get_contents]: http://php.net/file_get_contents
[require_once]: http://php.net/require_once
[fopen]: http://www.php.net/fopen
[stream-get-contents]: http://www.php.net/stream_get_contents
[install-git]: http://git-scm.com/book/en/Getting-Started-Installing-Git
[composer-phar]: http://getcomposer.org/composer.phar
