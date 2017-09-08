---
layout: post
title: Command Line Compression Tools
subtitle: Windows and Linux
---

There are several scenarios where is very useful to know how to compress/decompress files via command line.  
One example is the configuration of a continuous integration plan. We could need to compress directories to expose them as an artifact.  
Here is a short list of command line options for Windows and Linux and a third-party option using 7zip.    

## Windows Built In

### Powershell (Windows 8+, Server 2012+)

Compress

    Add-Type -A System.IO.Compression.FileSystem
    [IO.Compression.ZipFile]::CreateFromDirectory([folderPath], [zipFilePath])

> folderPath: path to the folder to be compressed   
> zipFilePath: destination zip file 

Example:

    Add-Type -A System.IO.Compression.FileSystem
    [IO.Compression.ZipFile]::CreateFromDirectory('C:\Source\MyProject', 'C:\source\my-project.zip')

Extract

    Add-Type -A System.IO.Compression.FileSystem
    [IO.Compression.ZipFile]::ExtractToDirectory([zipFilePath], [folderPath])

> zipFilePath: source zip file 
> folderPath: path to a folder where the zip content will be extracted   

Example:

    Add-Type -A System.IO.Compression.FileSystem
    [IO.Compression.ZipFile]::ExtractToDirectory([zipFilePath], [folderPath])

Extra options:

    [IO.Compression.ZipFile]::CreateFromDirectory([folderPath], [zipFilePath], [compressionLevel], [includeBaseDirectory], [encoding])

> compressionLevel: Fastest, NoCompression or Optimal  
> includeBaseDirectory: true, false  
> encoding: ASCII, Unicode, UTF32, UTF8, etc.  

For example:

    [IO.Compression.ZipFile]::CreateFromDirectory('C:\Source\MyProject', 'C:\source\my-project.zip', [IO.Compression.CompressionLevel]::Fastest, 1, [Text.Encoding]::UTF8)

Note: If you get a `*.ps1 cannot be loaded because the execution of scripts is disabled on this system...`

You will have to enable it first.You can view the execution policy status running:

    Get-ExecutionPolicy
    
You can find an explanation of the different execution policies in the following link:[https://technet.microsoft.com/en-us/library/ee176961.aspx](https://technet.microsoft.com/en-us/library/ee176961.aspx)

You can configure it with RemoteSigned or for a more secure option you can scope it to the actual user:

    Set-ExecutionPolicy RemoteSigned -Scope CurrentUser

## Linux

### Tar

Compress

    tar -czf my-project.tar.gz ~\projects\my-project

If for wherever reason you have a C:\Windows\temp folder in your files, you can exclude it with: 

    tar -czf my-project.tar.gz ~\projects\my-project --exclude="C:\\\Windows\\\temp"

Extract 

    tar -xzf my-project.tar.gz ~\projects\my-project

## Third Party

### 7z

Compress

    7z.exe a -r -y {output-file.[zip|7z|etc]} [path-to-folder]

> a: Add files to archive  
> r: Recurse subdirectories  
> y: Assume Yes in all queries 

Example:

    7z.exe a -r -y "C:\Projects\my-project.zip" "C:\Projects\My-Project"

Example excluding a folder:

    7z.exe a -r -y -xr!.git "C:\Projects\my-project.zip" "C:\Projects\My-Project"

Extract

    7z.exe x -o[folder-name] {file-name.[zip|7z|etc]}

Example 

    7z.exe x "C:\Projects\my-project.zip" -o"C:\Projects\My-Project"