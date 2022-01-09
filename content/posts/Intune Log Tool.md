---
title: "Intune Log Tool"
date: 2022-01-08T20:44:59+01:00
draft: False
---
![Example image](/thomassteendijk/image/Pastedimage20220104160652.png)
![<Pasted image 20220104155558.png>](</thomassteendijk/image/Pasted image 20220104155558.png>)


# Intro
Intune can give a command to the client to collect log files and upload these to cloud
These log files can then be downloaded from endpoint.microsoft.com and locally examined.

# Collecting log files
In endpoint manager go to devices > all devices > find your device > collect diagnostics > "Yes"

![Pasted image 20220104155258.png](/thomassteendijk/image/Pasted image 20220104155258.png)

# Downloading the logs
after the logs are uploaded they wil be come available for download (this normally takes 5-20 min depending on the clients internet speed)

![Pasted image 20220104155558.png](/thomassteendijk/image/Pasted image 20220104155558.png)

![Pasted image 20220104155919.png](/thomassteendijk/image/Pasted image 20220104155919.png)

# Using the log
By default the log comes in a zip and is hard to read
![Pasted image 20220104160019.png](/thomassteendijk/image/Pasted image 20220104160019.png)

For this is the script below to transform it into something usefull

## Expand-IntuneLogFile.ps1
```Powershell

<#
.Synopsis
   Expands intunes logs to a better readable format
.DESCRIPTION
   Takes the intune log file, unarchive it, read the result.xml and reforms the folder structure.
.EXAMPLE
    Create a expanded log in the same folder as the source file
   Expand-IntuneLogFile -IntuneLogFile "C:\Users\Thomas\Desktop\DiagLogs-LAP023985-20220103T192232Z.zip"
.EXAMPLE
    Create a expanded log in the ExportLocation folder
   Expand-IntuneLogFile -IntuneLogFile "C:\Users\Thomas\Desktop\DiagLogs-LAP023985-20220103T192232Z.zip" -ExportLocation C:\temp
.EXAMPLE
    Create a expanded log in the same folder as the source file by copying the path of a log file and running the powershell script.
    As the cmdbelow at the end of the script 
    allows the following work flow: 
    DiagLogs-LAP023985-20220103T192232Z.zip > right click > copy as path
    Expand-IntuneLogFile.ps1 > run with powershell
   
   Command for the end of the script:
   Expand-IntuneLogFile -IntuneLogFile (Get-Clipboard).Replace("`"","")

#>
function Expand-IntuneLogFile
{
    [CmdletBinding()]
    [Alias()]
    [OutputType([int])]
    Param
    (
        # Param1 help description
        [Parameter(Mandatory=$true,
                   ValueFromPipelineByPropertyName=$true,
                   Position=0)]
        [ValidateScript({try 
            {
                Get-Item $_ -ErrorAction Stop
            } catch [System.Management.Automation.ItemNotFoundException] {
                Throw [System.Management.Automation.ItemNotFoundException] "${_} Check the path in `$IntuneLogFile"
            }})]
        [string]
        $IntuneLogFile,

        # Param2 help description
        $ExportLocation = (get-item $IntuneLogFile).Directory.FullName
    )

    Begin
    {
        #region Formatting Choices 
        $flatFileNameTemplate = '({0:D2}) {1} {2}' 
        $maxLengthForInputTextPassedToOutput = 80 
        #endregion 
    }
    Process
    {
        #step 1: expand archive to $loglocation
        Unblock-File -Path $IntuneLogFile
        $diagnosticArchiveTempUnzippedPath = Join-Path $ExportLocation "$((Get-Item -Path $IntuneLogFile).BaseName)_expanded" 
        if(-not(Test-Path $diagnosticArchiveTempUnzippedPath)){New-Item $diagnosticArchiveTempUnzippedPath -ItemType Directory |Out-Null} 
        $reformattedArchivePath            = Join-Path $ExportLocation "$((Get-Item -Path $IntuneLogFile).BaseName)" 
        if(-not (Test-Path $reformattedArchivePath)){New-Item $reformattedArchivePath -ItemType Directory |Out-Null} 
        Expand-Archive -Path $IntuneLogFile -DestinationPath $diagnosticArchiveTempUnzippedPath

        #step 2: format folders
        $resultElements = ([xml](Get-Content -Path (Join-Path -Path $diagnosticArchiveTempUnzippedPath -ChildPath "results.xml"))).Collection.ChildNodes #| Foreach-Object{ $_ } 
        $n = 1 
 
        # only process supported directives 
        $supportedDirectives = @('Command', 'Events', 'FoldersFiles', 'RegistryKey') 
        foreach( $element in $resultElements) { 
          # only process supported directives, skip unsupported ones 
          if(!$supportedDirectives.Contains($element.Name)) { continue } 
 
          $directiveNumber       = $n 
          $n++ 
          $directiveType         = $element.Name 
          $directiveStatus       = [int]$element.Attributes.ItemOf('HRESULT').psbase.Value 
          $directiveUserInputRaw = $element.InnerText 
 
          # trim the path to only include the actual command - not the full path 
          if ($element.Name -eq 'Command') { 
            $lastIndexOfSlash = $directiveUserInputRaw.LastIndexOf('\'); 
            $directiveUserInputRaw = $directiveUserInputRaw.substring($lastIndexOfSlash+1); 
          } 
 
          $directiveUserInputFileNameCompatible = $directiveUserInputRaw -replace '[\\|/\[\]<>\:"\?\*%\.\s]','_' 
          $directiveUserInputTrimmed            = $directiveUserInputFileNameCompatible.substring(0, [System.Math]::Min($maxLengthForInputTextPassedToOutput, $directiveUserInputFileNameCompatible.Length)) 
          $directiveSummaryString               = $flatFileNameTemplate -f $directiveNumber,$directiveType,$directiveUserInputTrimmed 
          $directiveOutputFolder                = Join-Path -Path $diagnosticArchiveTempUnzippedPath -ChildPath $directiveNumber 
          $directiveOutputFiles                 = Get-ChildItem -Path $directiveOutputFolder -File 
          foreach( $file in $directiveOutputFiles) { 
            $leafSummaryString = $directiveSummaryString,$file.Name -join ' ' 
            Copy-Item $file.FullName -Destination (Join-Path -Path $reformattedArchivePath -ChildPath $leafSummaryString) 
          } 
        } 
        #endregion  
        Remove-Item -Path $diagnosticArchiveTempUnzippedPath -Force -Recurse 
        Get-Item $reformattedArchivePath
        
    }
}

```

this script can be used in the following methodes

## With the script loaded 

```Powershell
Expand-IntuneLogFile -IntuneLogFile "C:\Users\Thomas\Desktop\DiagLogs-LAP023985-20220103T192232Z.zip"
```

this will unpack the DiagLogs.zip to the same folder as the zip itself

```Powershell
Expand-IntuneLogFile -IntuneLogFile "C:\Users\Thomas\Desktop\DiagLogs-LAP023985-20220103T192232Z.zip" -ExportLocation C:\temp
```

This wil unpack the Dialogs.zip to the selected folder in -ExportLocation (in this example C:\temp) if the folder path does not exist one will be created. 


## Without the script loaded
You can also add the following as the last line in the script to use the "Run with powershell" option. 

```Powershell
Expand-IntuneLogFile -IntuneLogFile (Get-Clipboard).Replace("`"","")
```

Now you can copy (shift + right click > copy as path) the file path:
![Pasted image 20220104160652.png](/thomassteendijk/image/Pasted image 20220104160652.png)

and run the script with powershell
![Pasted image 20220104160750.png](/thomassteendijk/image/Pasted image 20220104160750.png)

![Pasted image 20220104160856.png](/thomassteendijk/image/Pasted image 20220104160856.png)

Or my personal favorite: 
```Powershell
Expand-IntuneLogFile -IntuneLogFile (Get-Clipboard).Replace("`"","") -ExportLocation "C:\IntuneLogStorage"
```

with this I can just copy as path, run the script and all my logs are collected in one location for future reference or easy to clean up.
![Pasted image 20220104161101.png](/thomassteendijk/image/Pasted image 20220104161101.png)

```Powershell
function Expand-IntuneLogFile
{
    [CmdletBinding()]
    [Alias()]
    [OutputType([int])]
    Param
    (
        # Param1 help description
        [Parameter(Mandatory=$true,
                   ValueFromPipelineByPropertyName=$true,
                   Position=0)]
        [ValidateScript({try 
            {
                Get-Item $_ -ErrorAction Stop
            } catch [System.Management.Automation.ItemNotFoundException] {
                Throw [System.Management.Automation.ItemNotFoundException] "${_} Check the path in `$IntuneLogFile"
            }})]
        [string]
        $IntuneLogFile,

        # Param2 help description
        $ExportLocation = (get-item $IntuneLogFile).Directory.FullName
    )

    Begin
    {
        #region Formatting Choices 
        $flatFileNameTemplate = '{0}\{1}' 
        $maxLengthForInputTextPassedToOutput = 80 
        #endregion 
    }
    Process
    {
        #step 1: expand archive to $loglocation
        Unblock-File -Path $IntuneLogFile
        $diagnosticArchiveTempUnzippedPath = Join-Path $ExportLocation "$((Get-Item -Path $IntuneLogFile).BaseName)_expanded" 
        if(-not(Test-Path $diagnosticArchiveTempUnzippedPath)){New-Item $diagnosticArchiveTempUnzippedPath -ItemType Directory |Out-Null} 
        $reformattedArchivePath            = Join-Path $ExportLocation "$((Get-Item -Path $IntuneLogFile).BaseName)" 
        #if(-not (Test-Path $reformattedArchivePath)){New-Item $reformattedArchivePath -ItemType Directory |Out-Null} 
        Expand-Archive -Path $IntuneLogFile -DestinationPath $diagnosticArchiveTempUnzippedPath

        #step 2: format folders
        $resultElements = ([xml](Get-Content -Path (Join-Path -Path $diagnosticArchiveTempUnzippedPath -ChildPath "results.xml"))).Collection.ChildNodes #| Foreach-Object{ $_ } 
        $n = 1 
 
        # only process supported directives 
        $supportedDirectives = @('Command', 'Events', 'FoldersFiles', 'RegistryKey') 
        foreach( $element in $resultElements) { 
          # only process supported directives, skip unsupported ones 
          if(!$supportedDirectives.Contains($element.Name)) { continue } 
 
          $directiveNumber       = $n 
          $n++ 
          $directiveType         = $element.Name 
          $directiveStatus       = [int]$element.Attributes.ItemOf('HRESULT').psbase.Value 
          $directiveUserInputRaw = $element.InnerText 
 
          # trim the path to only include the actual command - not the full path 
          if ($element.Name -eq 'Command') { 
            $lastIndexOfSlash      = $directiveUserInputRaw.LastIndexOf('\'); 
            $directiveUserInputRaw = $directiveUserInputRaw.substring($lastIndexOfSlash+1); 
          } 
 
          $directiveUserInputFileNameCompatible = $directiveUserInputRaw -replace '[\\|/\[\]<>\:"\?\*%\.\s]','_' 
          $directiveUserInputTrimmed            = $directiveUserInputFileNameCompatible.substring(0, [System.Math]::Min($maxLengthForInputTextPassedToOutput, $directiveUserInputFileNameCompatible.Length)) 
          $directiveSummaryString               = $flatFileNameTemplate -f $directiveType,$directiveUserInputTrimmed 
          $directiveOutputFolder                = Join-Path -Path $diagnosticArchiveTempUnzippedPath -ChildPath $directiveNumber 
          $directiveOutputFiles                 = Get-ChildItem -Path $directiveOutputFolder -File 
          foreach( $file in $directiveOutputFiles) { 
            $leafSummaryString = $directiveSummaryString,$file.Name -join ' ' 
            $MainFolder = $(Join-Path $reformattedArchivePath $directiveSummaryString).Replace("-","\").Replace("_"," ").Replace("\ ","\").Replace("   \","\").Replace("  \","\").Replace(" \","\").Replace("   "," ").Replace("  "," ").TrimEnd().Replace(" ","\")
            #if ($MainFolder[-1] = " ") {$MainFolder = $MainFolder.Substring(0,$MainFolder.Length - 2)}
            if (!(Test-Path $MainFolder)) { new-item -ItemType Directory -Path $MainFolder | Out-Null}
            Copy-Item $file.FullName -Destination (Join-Path -Path $MainFolder -ChildPath $file.Name) 
          } 
        } 
        #endregion  
        Remove-Item -Path $diagnosticArchiveTempUnzippedPath -Force -Recurse 
        #Get-Item $reformattedArchivePath
        
    }
}

```
