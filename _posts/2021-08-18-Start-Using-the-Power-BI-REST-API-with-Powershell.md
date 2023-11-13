---
layout: post
title:  Start Using the Power BI REST API with Powershell
categories: [Power BI, Powershell, Business Intelligence]
---

Earlier today, I used for the first time the Power BI REST API to access some reports in my company's workspace. This API can be used with Windows PowerShell. After spending a couple hours familiarizing myself with both PowerShell and the API, I think I've finally got the hang of it.

If you've never used the Power BI REST API previously with PowerShell previously, you first need to download the module onto your system. Enter the following into the PowerShell console:

```powershell
# A REST API is simply an interface for interacting with a server. Full details on
# the API functionality can be found at: https://docs.microsoft.com/en-us/rest/api/power-bi/

Install-Module -Name MicrosoftPowerBIMgmt # please type this line into your PowerShell console
```
<br>

Next, I will demonstrate how to quickly interact with the API and your published content. If you haven't yet, open up PowerShell ISE, an IDE for PowerShell Scripting (though, I am sure VS Code would work just fine, too). I encourage you to run each line of code independently (not all at once) and to examine the structure and contents of each variable. And of course, don't forget to save your script when finished.

Here, you will be able to access metadata on your reports:

```powershell
Login-Power BI # This initial line will open up a login screen for the Power BI service

<# 
Ensure you find the group ID for your content. This is located in the URL of your workspace
E.g., app.powerbi.com/groups/<YOUR-GROUP-ID>/list
Also note how in PowerShell, variables are initialized and called using the dollar-sign 
#>
$result = Invoke-PowerBIRestMethod - URL"https://api.powerbi.com/v1.0/myorg/groups/<YOUR-GROUP-ID>/datasets" -Method Get

<# 
Unlike in R where piping is performed with %>% when using the Tidyverse
and |> in base R, PowerShell uses an actual pipe character to transfer the
contents of $result to the function 'ConvertFrom-Json'
#>
$workspaceContents = $result | ConvertFrom-Json 

# For testing purposes, let's try to extract the contents of the first report listed.
# type this into the console and then call $firstWorkspace
$firstWorkspace = $workspaceContents.value[0]

# Of course, it's bad practice to specify an object by referencing a direct location.
# What if that changes? Let's set up the basis of a for-loop by getting the number
# of reports stored in our workspace:
$m = $workspaceContents.value | measure

<# 
Now we can create our for-loop. Note the similarities of a PowerShell 
for-loop structure with Java syntax. '-lt' is a less-than operator here,
specifying to iterate i (beginning at 0) with increments of 1 until it 
reaches less than the value of $m.Count
#>
for ($i=0; $i -lt $m.Count; $i++){
    if ($workspaceContents.value[$i].name -eq "My Geospatial Report"){ # '-eq' is the equals operator
        $desired_report = $workspaceContents.value[$i]
        Write-Host "The report was found!" # this is a print-to-console operation
        break # this stops the loop at the current iteration
    }
}

# call the report metadata:
$desired_report
```

I may post more tutorials and code snippets on Power BI API implementations in the future. Until then, I would strongly recommend watching [Guy in a Cube](https://www.youtube.com/@GuyInACube){:target="_blank"} tutorials on YouTube. The channel has some great content for all things Power BI, including implementation of the REST API. Thanks for stopping by, and happy coding!