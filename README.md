POST {batchUrl}/pools?api-version=2024-07-01.20.0

POST {batchUrl}/jobs?api-version=2024-07-01.20.0

GET {batchUrl}/nodecounts?api-version=2024-07-01.20.0
GET account.region.batch.azure.com/nodecounts?api-version=2024-07-01.20.0
﻿using Azure.Storage;
using Azure.Storage.Blobs;
using Azure.Storage.Sas;
using Microsoft.Azure.Batch;
using Microsoft.Azure.Batch.Auth;
using Microsoft.Azure.Batch.Common;
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;

namespace BatchDotNetQuickstart
{
    public class Program
    {
        // Update the Batch and Storage account credential strings below with the values unique to your accounts.
        // These are used when constructing connection strings for the Batch and Storage client objects.

        // Batch account credentials
        private const string BatchAccountName = "";
        private const string BatchAccountKey = "";
        private const string BatchAccountUrl = "";

        // Storage account credentials
        private const string StorageAccountName = "";
        private const string StorageAccountKey = "";

        // Batch resource settings
        private const string PoolId = "DotNetQuickstartPool";
        private const string JobId = "DotNetQuickstartJob";
        private const int PoolNodeCount = 2;
        private const string PoolVMSize = "STANDARD_D1_V2";

        static void Main()
        {
            if (string.IsNullOrEmpty(BatchAccountName) ||
                string.IsNullOrEmpty(BatchAccountKey) ||
                string.IsNullOrEmpty(BatchAccountUrl) ||
                string.IsNullOrEmpty(StorageAccountName) ||
                string.IsNullOrEmpty(StorageAccountKey))
            {
                throw new InvalidOperationException("One or more account credential strings have not been populated. Please ensure that your Batch and Storage account credentials have been specified.");
            }

            try
            {
                Console.WriteLine("Sample start: {0}", DateTime.Now);
                Console.WriteLine();
                var timer = new Stopwatch();
                timer.Start();

                // Create the blob client, for use in obtaining references to blob storage containers
                var blobServiceClient = GetBlobServiceClient(StorageAccountName, StorageAccountKey);

                // Use the blob client to create the input container in Azure Storage 
                const string inputContainerName = "input";
                var containerClient = blobServiceClient.GetBlobContainerClient(inputContainerName);
                containerClient.CreateIfNotExistsAsync().Wait();

                // The collection of data files that are to be processed by the tasks
                List<string> inputFilePaths = new()
                {
                    "taskdata0.txt",
                    "taskdata1.txt",
                    "taskdata2.txt"
                };

                // Upload the data files to Azure Storage. This is the data that will be processed by each of the tasks that are
                // executed on the compute nodes within the pool.
                var inputFiles = new List<ResourceFile>();

                foreach (var filePath in inputFilePaths)
                {
                    inputFiles.Add(UploadFileToContainer(containerClient, inputContainerName, filePath));
                }

                // Get a Batch client using account creds
                var cred = new BatchSharedKeyCredentials(BatchAccountUrl, BatchAccountName, BatchAccountKey);

                using BatchClient batchClient = BatchClient.Open(cred);
                Console.WriteLine("Creating pool [{0}]...", PoolId);

                // Create a Windows Server image, VM configuration, Batch pool
                ImageReference imageReference = CreateImageReference();
                VirtualMachineConfiguration vmConfiguration = CreateVirtualMachineConfiguration(imageReference);
                CreateBatchPool(batchClient, vmConfiguration);

                // Create a Batch job
                Console.WriteLine("Creating job [{0}]...", JobId);

                try
                {
                    CloudJob job = batchClient.JobOperations.CreateJob();
                    job.Id = JobId;
                    job.PoolInformation = new PoolInformation { PoolId = PoolId };
                    job.Commit();
                }
                catch (BatchException be)
                {
                    // Accept the specific error code JobExists as that is expected if the job already exists
                    if (be.RequestInformation?.BatchError?.Code == BatchErrorCodeStrings.JobExists)
                    {
                        Console.WriteLine("The job {0} already existed when we tried to create it", JobId);
                    }
                    else
                    {
                        throw; // Any other exception is unexpected
                    }
                }

                // Create a collection to hold the tasks that we'll be adding to the job
                Console.WriteLine("Adding {0} tasks to job [{1}]...", inputFiles.Count, JobId);
                var tasks = new List<CloudTask>();

                // Create each of the tasks to process one of the input files. 
                for (int i = 0; i < inputFiles.Count; i++)
                {
                    string taskId = string.Format("Task{0}", i);
                    string inputFilename = inputFiles[i].FilePath;
                    string taskCommandLine = string.Format("cmd /c type {0}", inputFilename);

                    var task = new CloudTask(taskId, taskCommandLine)
                    {
                        ResourceFiles = new List<ResourceFile> { inputFiles[i] }
                    };
                    tasks.Add(task);
                }

                // Add all tasks to the job.
                batchClient.JobOperations.AddTask(JobId, tasks);

                // Monitor task success/failure, specifying a maximum amount of time to wait for the tasks to complete.
                TimeSpan timeout = TimeSpan.FromMinutes(30);
                Console.WriteLine("Monitoring all tasks for 'Completed' state, timeout in {0}...", timeout);

                IEnumerable<CloudTask> addedTasks = batchClient.JobOperations.ListTasks(JobId);
                batchClient.Utilities.CreateTaskStateMonitor().WaitAll(addedTasks, TaskState.Completed, timeout);
                Console.WriteLine("All tasks reached state Completed.");

                // Print task output
                Console.WriteLine();
                Console.WriteLine("Printing task output...");

                IEnumerable<CloudTask> completedtasks = batchClient.JobOperations.ListTasks(JobId);
                foreach (CloudTask task in completedtasks)
                {
                    string nodeId = string.Format(task.ComputeNodeInformation.ComputeNodeId);
                    Console.WriteLine("Task: {0}", task.Id);
                    Console.WriteLine("Node: {0}", nodeId);
                    Console.WriteLine("Standard out:");
                    Console.WriteLine(task.GetNodeFile(Constants.StandardOutFileName).ReadAsString());
                }

                // Print out some timing info
                timer.Stop();
                Console.WriteLine();
                Console.WriteLine("Sample end: {0}", DateTime.Now);
                Console.WriteLine("Elapsed time: {0}", timer.Elapsed);

                // Clean up Storage resources
                containerClient.DeleteIfExistsAsync().Wait();
                Console.WriteLine("Container [{0}] deleted.", inputContainerName);

                // Clean up Batch resources (if the user so chooses)
                Console.WriteLine();
                Console.Write("Delete job? [yes] no: ");
                string response = Console.ReadLine().ToLower();
                if (response != "n" && response != "no")
                {
                    batchClient.JobOperations.DeleteJob(JobId);
                }

                Console.Write("Delete pool? [yes] no: ");
                response = Console.ReadLine().ToLower();
                if (response != "n" && response != "no")
                {
                    batchClient.PoolOperations.DeletePool(PoolId);
                }
            }
            catch(Exception e) 
            {
                Console.WriteLine(e.Message);
            }
            finally
            {
                Console.WriteLine();
                Console.WriteLine("Sample complete, hit ENTER to exit...");
                Console.ReadLine();
            }
        }

        private static void CreateBatchPool(BatchClient batchClient, VirtualMachineConfiguration vmConfiguration)
        {
            try
            {
                CloudPool pool = batchClient.PoolOperations.CreatePool(
                    poolId: PoolId,
                    targetDedicatedComputeNodes: PoolNodeCount,
                    virtualMachineSize: PoolVMSize,
                    virtualMachineConfiguration: vmConfiguration);

                pool.Commit();
            }
            catch (BatchException be)
            {
                // Accept the specific error code PoolExists as that is expected if the pool already exists
                if (be.RequestInformation?.BatchError?.Code == BatchErrorCodeStrings.PoolExists)
                {
                    Console.WriteLine("The pool {0} already existed when we tried to create it", PoolId);
                }
                else
                {
                    throw; // Any other exception is unexpected
                }
            }
        }

        private static VirtualMachineConfiguration CreateVirtualMachineConfiguration(ImageReference imageReference)
        {
            return new VirtualMachineConfiguration(
                imageReference: imageReference,
                nodeAgentSkuId: "batch.node.windows amd64");
        }

        private static ImageReference CreateImageReference()
        {
            return new ImageReference(
                publisher: "MicrosoftWindowsServer",
                offer: "WindowsServer",
                sku: "2016-datacenter-smalldisk",
                version: "latest");
        }

        /// <summary>
        /// Creates a blob client
        /// </summary>
        /// <param name="storageAccountName">The name of the Storage Account</param>
        /// <param name="storageAccountKey">The key of the Storage Account</param>
        /// <returns></returns>
        private static BlobServiceClient GetBlobServiceClient(string storageAccountName, string storageAccountKey)
        {
            var sharedKeyCredential = new StorageSharedKeyCredential(storageAccountName, storageAccountKey);
            string blobUri = "https://" + storageAccountName + ".blob.core.windows.net";

            var blobServiceClient = new BlobServiceClient(new Uri(blobUri), sharedKeyCredential);
            return blobServiceClient;
        }

        /// <summary>
        /// Uploads the specified file to the specified Blob container.
        /// </summary>
        /// <param name="containerClient">A <see cref="BlobContainerClient"/>.</param>
        /// <param name="containerName">The name of the blob storage container to which the file should be uploaded.</param>
        /// <param name="filePath">The full path to the file to upload to Storage.</param>
        /// <returns>A ResourceFile instance representing the file within blob storage.</returns>
        private static ResourceFile UploadFileToContainer(BlobContainerClient containerClient, string containerName, string filePath, string storedPolicyName = null)
        {
            Console.WriteLine("Uploading file {0} to container [{1}]...", filePath, containerName);
            string blobName = Path.GetFileName(filePath);
            filePath = Path.Combine(Environment.CurrentDirectory, filePath);

            var blobClient = containerClient.GetBlobClient(blobName);
            blobClient.Upload(filePath, true);

            // Set the expiry time and permissions for the blob shared access signature. 
            // In this case, no start time is specified, so the shared access signature 
            // becomes valid immediately
            // Check whether this BlobContainerClient object has been authorized with Shared Key.
            if (blobClient.CanGenerateSasUri)
            {
                // Create a SAS token
                var sasBuilder = new BlobSasBuilder()
                {
                    BlobContainerName = containerClient.Name,
                    BlobName = blobClient.Name,
                    Resource = "b"
                };

                if (storedPolicyName == null)
                {
                    sasBuilder.ExpiresOn = DateTimeOffset.UtcNow.AddHours(1);
                    sasBuilder.SetPermissions(BlobContainerSasPermissions.Read);
                }
                else
                {
                    sasBuilder.Identifier = storedPolicyName;
                }

                var sasUri = blobClient.GenerateSasUri(sasBuilder).ToString();
                return ResourceFile.FromUrl(sasUri, filePath);
            }
            else
            {
                throw new InvalidOperationException("BlobClient must be authorized with shared key credentials to create a service SAS.");
            }
        }
    }
}


Provide your account information
The app needs to use your Batch and Storage account names, account key values, and Batch account endpoint. You can get this information from the Azure portal, Azure APIs, or command-line tools.

To get your account information from the Azure portal:

From the Azure Search bar, search for and select your Batch account name.
On your Batch account page, select Keys from the left navigation.
On the Keys page, copy the following values:
Batch account
Account endpoint
Primary access key
Storage account name
Key1
Navigate to your downloaded batch-dotnet-quickstart folder and edit the credential strings in Program.cs to provide the values you copied:

C#

Copy
// Batch account credentials
private const string BatchAccountName = "<batch account>";
private const string BatchAccountKey  = "<primary access key>";
private const string BatchAccountUrl  = "<account endpoint>";

// Storage account credentials
private const string StorageAccountName = "<storage account name>";
private const string StorageAccountKey  = "<key1>
 Important

Exposing account keys in the app source isn't recommended for Production usage. You should restrict access to credentials and refer to them in your code by using variables or a configuration file. It's best to store Batch and Storage account keys in Azure Key Vault.

Build and run the app and view output
To see the Batch workflow in action, build and run the application in Visual Studio. You can also use the command line dotnet build and dotnet run commands.

In Visual Studio:

Open the BatchDotNetQuickstart.sln file, right-click the solution in Solution Explorer, and select Build. If prompted, use NuGet Package Manager to update or restore NuGet packages.

Once the build completes, select BatchDotNetQuickstart in the top menu bar to run the app.

Typical run time with the default configuration is approximately five minutes. Initial pool node setup takes the most time. To rerun the job, delete the job from the previous run, but don't delete the pool. On a preconfigured pool, the job completes in a few seconds.

The app returns output similar to the following example:

Output

Copy
Sample start: 11/16/2022 4:02:54 PM

Container [input] created.
Uploading file taskdata0.txt to container [input]...
Uploading file taskdata1.txt to container [input]...
Uploading file taskdata2.txt to container [input]...
Creating pool [DotNetQuickstartPool]...
Creating job [DotNetQuickstartJob]...
Adding 3 tasks to job [DotNetQuickstartJob]...
Monitoring all tasks for 'Completed' state, timeout in 00:30:00...
There's a pause at Monitoring all tasks for 'Completed' state, timeout in 00:30:00... while the pool's compute nodes start. As tasks are created, Batch queues them to run on the pool. As soon as the first compute node is available, the first task runs on the node. You can monitor node, task, and job status from your Batch account page in the Azure portal.

After each task completes, you see output similar to the following example:

Output

Copy
Printing task output.
Task: Task0
Node: tvm-2850684224_3-20171205t000401z
Standard out:
Batch processing began with mainframe computers and punch cards. Today it still plays a central role...
stderr:
...
Review the code
Review the code to understand the steps in the Azure Batch .NET Quickstart.

Create service clients and upload resource files
To interact with the storage account, the app uses the Azure Storage Blobs client library for .NET to create a BlobServiceClient.

C#

Copy
var sharedKeyCredential = new StorageSharedKeyCredential(storageAccountName, storageAccountKey);
string blobUri = "https://" + storageAccountName + ".blob.core.windows.net";

var blobServiceClient = new BlobServiceClient(new Uri(blobUri), sharedKeyCredential);
return blobServiceClient;
The app uses the blobServiceClient reference to create a container in the storage account and upload data files to the container. The files in storage are defined as Batch ResourceFile objects that Batch can later download to the compute nodes.

C#

Copy
List<string> inputFilePaths = new()
{
    "taskdata0.txt",
    "taskdata1.txt",
    "taskdata2.txt"
};

var inputFiles = new List<ResourceFile>();

foreach (var filePath in inputFilePaths)
{
    inputFiles.Add(UploadFileToContainer(containerClient, inputContainerName, filePath));
}
The app creates a BatchClient object to create and manage Batch pools, jobs, and tasks. The Batch client uses shared key authentication. Batch also supports Microsoft Entra authentication.

C#

Copy
var cred = new BatchSharedKeyCredentials(BatchAccountUrl, BatchAccountName, BatchAccountKey);

 using BatchClient batchClient = BatchClient.Open(cred);
...
Create a pool of compute nodes
To create a Batch pool, the app uses the BatchClient.PoolOperations.CreatePool method to set the number of nodes, VM size, and pool configuration. The following VirtualMachineConfiguration object specifies an ImageReference to a Windows Server Marketplace image. Batch supports a wide range of Windows Server and Linux Marketplace OS images, and also supports custom VM images.

The PoolNodeCount and VM size PoolVMSize are defined constants. The app creates a pool of two Standard_A1_v2 nodes. This size offers a good balance of performance versus cost for this quickstart.

The Commit method submits the pool to the Batch service.

C#

Copy

private static VirtualMachineConfiguration CreateVirtualMachineConfiguration(ImageReference imageReference)
{
    return new VirtualMachineConfiguration(
        imageReference: imageReference,
        nodeAgentSkuId: "batch.node.windows amd64");
}

private static ImageReference CreateImageReference()
{
    return new ImageReference(
        publisher: "MicrosoftWindowsServer",
        offer: "WindowsServer",
        sku: "2016-datacenter-smalldisk",
        version: "latest");
}

private static void CreateBatchPool(BatchClient batchClient, VirtualMachineConfiguration vmConfiguration)
{
    try
    {
        CloudPool pool = batchClient.PoolOperations.CreatePool(
            poolId: PoolId,
            targetDedicatedComputeNodes: PoolNodeCount,
            virtualMachineSize: PoolVMSize,
            virtualMachineConfiguration: vmConfiguration);

        pool.Commit();
    }
...

Create a Batch job
A Batch job is a logical grouping of one or more tasks. The job includes settings common to the tasks, such as priority and the pool to run tasks on.

The app uses the BatchClient.JobOperations.CreateJob method to create a job on your pool. The Commit method submits the job to the Batch service. Initially the job has no tasks.

C#

Copy
try
{
    CloudJob job = batchClient.JobOperations.CreateJob();
    job.Id = JobId;
    job.PoolInformation = new PoolInformation { PoolId = PoolId };

    job.Commit();
}
...
Create tasks
Batch provides several ways to deploy apps and scripts to compute nodes. This app creates a list of CloudTask input ResourceFile objects. Each task processes an input file by using a CommandLine property. The Batch command line is where you specify your app or script.

The command line in the following code runs the Windows type command to display the input files. Then, the app adds each task to the job with the AddTask method, which queues the task to run on the compute nodes.

C#

Copy
for (int i = 0; i < inputFiles.Count; i++)
{
    string taskId = String.Format("Task{0}", i);
    string inputFilename = inputFiles[i].FilePath;
    string taskCommandLine = String.Format("cmd /c type {0}", inputFilename);

    var task = new CloudTask(taskId, taskCommandLine)
    {
        ResourceFiles = new List<ResourceFile> { inputFiles[i] }
    };
    tasks.Add(task);
}

batchClient.JobOperations.AddTask(JobId, tasks);
View task output
The app creates a TaskStateMonitor to monitor the tasks and make sure they complete. When each task runs successfully, its output writes to stdout.txt. The app then uses the CloudTask.ComputeNodeInformation property to display the stdout.txt file for each completed task.

C#

Copy
foreach (CloudTask task in completedtasks)
{
    string nodeId = String.Format(task.ComputeNodeInformation.ComputeNodeId);
    Console.WriteLine("Task: {0}", task.Id);
    Console.WriteLine("Node: {0}", nodeId);
    Console.WriteLine("Standard out:");
    Console.WriteLine(task.GetNodeFile(Constants.StandardOutFileName).ReadAsString());
}

Using all above codes and all above inputs and all above reference can you give full and proper code for this

Azure Cloudtasks never finish, status is always active
Asked yesterday
Modified today
Viewed 43 times
 Part of Microsoft Azure Collective

Report this ad
Vote

I'm new to Azure Batch. I'm using a .NET 8 console app to execute a remote application for each file in a blob container, I'm sending as an argument the file name.

I tried 2 different command lines, not sure if the problem is there - any ideas?

// Azure Blob Storage details
using Azure.Storage;
using Azure.Storage.Blobs;
using Microsoft.Azure.Batch;
using Microsoft.Azure.Batch.Auth;
using Microsoft.Azure.Batch.Common;
using System.Diagnostics;

// Batch account credentials
const string BatchAccountName = "";
const string BatchAccountKey = "";
const string BatchAccountUrl = "";

// Storage account credentials
const string StorageAccountName = "";
const string StorageAccountKey = "";
const string ContainerName = "";
const string ExeContainerName = "";

// Batch resource settings
const string PoolId = "";
const string JobId = "";
const int PoolNodeCount = 2;
const string PoolVMSize = "";
const string AppUrl = $"";

var blobServiceClient = GetBlobServiceClient(StorageAccountName, StorageAccountKey);
var containerClient = blobServiceClient.GetBlobContainerClient(ContainerName);

TimeSpan timeout = TimeSpan.FromMinutes(2);
Console.WriteLine("Sample start: {0}", DateTime.Now);

try
{
    var timer = new Stopwatch();
    timer.Start();

    var blobNames = new List<string>();

    await foreach (var blobItem in containerClient.GetBlobsAsync())
    {
        blobNames.Add(blobItem.Name);
    }

    // Create Batch client
    // Get a Batch client using account creds
    var cred = new BatchSharedKeyCredentials(BatchAccountUrl, BatchAccountName, BatchAccountKey);

    BatchClient batchClient = BatchClient.Open(cred);
    Console.WriteLine("Creating pool [{0}]...", PoolId);

    // Create a Windows Server image, VM configuration, Batch pool
    ImageReference imageReference = CreateImageReference();
    VirtualMachineConfiguration vmConfiguration = CreateVirtualMachineConfiguration(imageReference);
    await CreateBatchPoolAsync(batchClient, vmConfiguration);

    // Create a Batch job
    Console.WriteLine("Creating job [{0}]...", JobId);

    try
    {
        CloudJob job = batchClient.JobOperations.CreateJob();
        job.Id = JobId;
        job.PoolInformation = new PoolInformation { PoolId = PoolId };
        job.Commit();
    }
    catch (BatchException be)
    {
        // Accept the specific error code JobExists as that is expected if the job already exists

        if (be.RequestInformation?.BatchError?.Code == BatchErrorCodeStrings.JobExists)
        {
            Console.WriteLine("The job {0} already existed when we tried to create it, please try again", JobId);
            batchClient.JobOperations.DeleteJob(JobId);
            return;
        }
        else
        {
            throw; // Any other exception is unexpected
        }
    }

    // Create a task for each blob
    var tasks = new List<CloudTask>();

    foreach (var blobName in blobNames)
    {
        // Command line: download blob and print content
        string blobSasUrl = containerClient.GetBlobClient(blobName).GenerateSasUri(Azure.Storage.Sas.BlobSasPermissions.Read, DateTimeOffset.UtcNow.AddHours(1)).ToString();
        //string commandLine = $"ConsoleApp.exe '{blobName}'";
        string workingDir = @"C:\myapp";
        string commandLine = $"cmd /c \"cd /d {workingDir} && ConsoleApp.exe '{blobName}'\"";

        CloudTask task = new CloudTask("myTask" + Guid.NewGuid(), commandLine)
        {
            ResourceFiles = new List<ResourceFile>
        {
                ResourceFile.FromUrl($"{AppUrl}/Azure.Core.dll","Azure.Core.dll"),
                ResourceFile.FromUrl($"{AppUrl}/Azure.Storage.Blobs.dll","Azure.Storage.Blobs.dll"),
                ResourceFile.FromUrl($"{AppUrl}/Azure.Storage.Common.dll","Azure.Storage.Common.dll"),
                ResourceFile.FromUrl($"{AppUrl}/ConsoleApp.deps.json","ConsoleApp.deps.json"),
                ResourceFile.FromUrl($"{AppUrl}/ConsoleApp.dll","ConsoleApp.dll"),
                ResourceFile.FromUrl($"{AppUrl}/ConsoleApp.exe","ConsoleApp.exe"),
                ResourceFile.FromUrl($"{AppUrl}/ConsoleApp.pdb","ConsoleApp.pdb"),
                ResourceFile.FromUrl($"{AppUrl}/ConsoleApp.runtimeconfig.json","ConsoleApp.runtimeconfig.json"),
                ResourceFile.FromUrl($"{AppUrl}/Microsoft.Bcl.AsyncInterfaces.dll","Microsoft.Bcl.AsyncInterfaces.dll"),
                ResourceFile.FromUrl($"{AppUrl}/System.ClientModel.dll","System.ClientModel.dll"),
                ResourceFile.FromUrl($"{AppUrl}/System.IO.Hashing.dll","System.IO.Hashing.dll"),
                ResourceFile.FromUrl($"{AppUrl}/System.Memory.Data.dll","System.Memory.Data.dll")
            }
        };
        tasks.Add(task);
    }

    // Add all tasks to the job.
    await batchClient.JobOperations.AddTaskAsync(JobId, tasks);
    //Console.WriteLine("Monitoring all tasks for 'Completed' state, timeout in {0}...", timeout);
    //var monitor = batchClient.Utilities.CreateTaskStateMonitor();
    ////  -- await monitor.WhenAll(tasks, TaskState.Completed, timeout);
    //var taskList = batchClient.JobOperations.ListTasks(JobId).ToList();
    //await monitor.WhenAll(taskList, TaskState.Completed, timeout);

    await WaitForTasksCompletion(batchClient, JobId);
    Console.WriteLine("All tasks reached state Completed.");

    // Print out some timing info
    timer.Stop();

    Console.WriteLine();
    Console.WriteLine("Sample end: {0}", DateTime.Now);
    Console.WriteLine("Elapsed time: {0}", timer.Elapsed);

    Console.WriteLine();
    Console.WriteLine("Tasks Info:");

    var tasksProcessed = batchClient.JobOperations.ListTasks(JobId).ToList();

    foreach (var task in tasksProcessed)
    {
        Console.WriteLine($"Task: {task.Id}");
        Console.WriteLine($"Url: {task.Url}");

        if (task.State.HasValue)
        {
            Console.WriteLine($"State: {task.State}");
        }

        if (task.ExecutionInformation != null)
        {
            Console.WriteLine($"ExitCode: {task.ExecutionInformation.ExitCode}");

            if (!string.IsNullOrEmpty(task.ExecutionInformation.FailureInformation?.Message))
            {
                Console.WriteLine($"Failure: {task.ExecutionInformation.FailureInformation.Message}");
            }
        }

        Console.WriteLine(new string('-', 40));
    }

    // Clean up Batch resources (if the user so chooses)
    Console.WriteLine();
    Console.Write("Delete job? [yes] no: ");
    string response = Console.ReadLine().ToLower();

    if (response != "n" && response != "no")
    {
        batchClient.JobOperations.DeleteJob(JobId);
    }

    Console.Write("Delete pool? [yes] no: ");
    response = Console.ReadLine().ToLower();

    if (response != "n" && response != "no")
    {
        batchClient.PoolOperations.DeletePool(PoolId);
    }
}
catch (Exception e)
{
    Console.WriteLine(e.Message);
}
finally
{
    Console.WriteLine();
    Console.WriteLine("Sample complete, hit ENTER to exit...");
    Console.ReadLine();
}

static async Task WaitForTasksCompletion(BatchClient batchClient, string jobId)
{
    var tasks = batchClient.JobOperations.ListTasks(jobId).Select(t => Task.Run(async () =>
    {
        while (t.State != TaskState.Completed)
        {
            await Task.Delay(TimeSpan.FromSeconds(5));
            t.Refresh();
        }
    })).ToArray();

    await Task.WhenAll(tasks);
    Console.WriteLine("All tasks completed!");
}

BlobServiceClient GetBlobServiceClient(string storageAccountName, string storageAccountKey)
{
    var sharedKeyCredential = new StorageSharedKeyCredential(storageAccountName, storageAccountKey);
    string blobUri = "https://" + storageAccountName + ".blob.core.windows.net";
    var blobServiceClient = new BlobServiceClient(new Uri(blobUri), sharedKeyCredential);
    return blobServiceClient;
}

static VirtualMachineConfiguration CreateVirtualMachineConfiguration(ImageReference imageReference)
{
    return new VirtualMachineConfiguration(
        imageReference: imageReference,
        nodeAgentSkuId: "batch.node.windows amd64");
}

static ImageReference CreateImageReference()
{
    return new ImageReference(
        publisher: "MicrosoftWindowsServer",
        offer: "WindowsServer",
        sku: "2016-datacenter-smalldisk",
        version: "latest");
}

static async Task CreateBatchPoolAsync(BatchClient batchClient, VirtualMachineConfiguration vmConfiguration)
{
    try
    {
        CloudPool pool = batchClient.PoolOperations.CreatePool(
        poolId: PoolId,
            targetDedicatedComputeNodes: PoolNodeCount,
            virtualMachineSize: PoolVMSize,
            virtualMachineConfiguration: vmConfiguration);

        await pool.CommitAsync();
    }
    catch (BatchException be)
    {
        // Accept the specific error code PoolExists as that is expected if the pool already exists
        if (be.RequestInformation?.BatchError?.Code == BatchErrorCodeStrings.PoolExists)
        {
            Console.WriteLine("The pool {0} already existed when we tried to create it, please try again", PoolId);
            batchClient.PoolOperations.DeletePool(PoolId);
            return;
        }
        else
        {
            throw; // Any other exception is unexpected
        }
    }
}
The error I see when I inspect/debug the url is:

{
  "odata.metadata":"https://some.url.batch.azure.com/$metadata#Microsoft.Azure.Batch.Protocol.Entities.Container.errors/@Element","code":"MissingRequiredQueryParameter","message":{
    "lang":"en-US","value":"A query parameter that's mandatory for this request is not specified.\nRequestId:a461fc08-156e-440d-b557-e65f65db0d41\nTime:2025-06-05T00:05:48.3590438Z"
  },"values":[
    {
      "key":"QueryParameterName","value":"api-version"
    }
  ]
}
As you can see, I have tried 2 different methods to detect when all tasks are finished, but they never change the status from active...

azure.net-8.0azure-batch
Share
Edit
Follow
Close
Flag
edited 22 hours ago
marc_s's user avatar
marc_s
758k184184 gold badges1.4k1.4k silver badges1.5k1.5k bronze badges
asked yesterday
David Alejandro García García's user avatar
David Alejandro García García
35611 gold badge44 silver badges1919 bronze badges

Add batchClient = BatchClient.Open(cred, new BatchClientOptions { BatchServiceVersion = new Version("2023-05-01.17.0") }); to explicitly set the api-versionand fix the missing query parameter error.@David Alejandro García García – 
Harshitha Bathini
 Commentedyesterday 

BatchClientOptions isnt listed in any of the 3 possible Open options..: Option1 pass a BatchSharedKeyCredentials. Option 2 pass a BatchTokenCredentias. Option 3 pass a BatchServiceClient – 
David Alejandro García García
 Commented10 hours ago 

I also tried the CSharp/GettingStarted/HelloWorld... 10 minutes timeout , whenall isnt working either... any ideas? github.com/Azure-Samples/azure-batch-samples/tree/master – 
David Alejandro García García
 Commented8 hours ago 

I see the same error: "code":"MissingRequiredQueryParameter","value":"A query parameter that's mandatory for this request is not specified."values":"key":"QueryParameterName","value":"api-version"
