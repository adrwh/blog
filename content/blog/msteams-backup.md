---
title: 'MS Teams Channel Conversation "Backup"'
date: 2020-01-01
description: "This is about how we backed up MS Teams Conversations"
ghissueid: 1
tags: ["msgraph","teams"]
draft: false
---

We use Microsoft Teams, and probably as most, create a new team when it was first released, however have now decided to move to a new Team.  This is all fine, but when we went looking for a "migration" pathway, there wasn't one.  Instead we settled on making a backup/copy for historical purposes.  

I went searching PowerShell for cmdlets to "backup" Teams and didn't find any. I then went searching the Microsoft Graph API and found the Teams API resources. One of the endpoints was called <span class="has-background-light has-text-danger">List Channel Messages (Link below)</span>, bingo!

https://docs.microsoft.com/en-us/graph/api/channel-list-messages

When you request the ms graph api, you need to provide a bearer token in the authorisation header. To get a token you need to setup an Azure AD Application, and go through one of the oauth flows.

With my app ready, I decided to use PowerShell to make the necessary authentication requests, get my access token, then makes requests to the Teams graph to list the channel messages.

Note that we have our access token we're ready to start poking the MS Graph API.

```powershell {linenos=true, linenostart=20}
# This function takes 3 parameters, then pulls messages from the MS Graph API
function GetMessages {
    param ([String]$team_id, [String]$channel_id, [String]$file_name)
    # The initial MS Graph request to the MS Teams List Channel Messages endpoint
    $uri = "https://graph.microsoft.com/beta/teams/$team_id/channels/$channel_id/messages"

    do {
        # Make your first request to the API
        # Note:  The $headers variable is a hash table with your Authorization header and Bearer token etc
        $res = Invoke-RestMethod -Uri $uri -Headers $headers -Method Get -SslProtocol:Tls12
        # Get the next URL
        $next = $res."@odata.nextLink"
        # If there is a next URL, update $uri ready for the next iteration
        if ($next) { $uri = $next }
        # Loop through each message
        $res.value | ForEach-Object {
            # Assign the message content to $raw so that we can strip out the text
            $raw = $_.body.content
            # This regex will strip out all HTML tags, just leaving the text content
            $msg = $raw -replace '<[^>]+>'
            # Assign the message created datetime
            $date = $_.createdDateTime
            # Build up the HTML rows for each message, we will use this later.  Note, you can use <table> or any other style you prefer
            $body += '<hr><div><time datetime="' + $date + '">' + $date + '</time> | ' + $msg + '</div>'
        }
        # I put a sleep in because i kept hitting the rate limitations without it
        Start-Sleep -Milliseconds 500
    # Keep looping as long as there is a Next URL
    } while ($null -ne $next)

# This is the HTML template.  We will "insert" the rows ($body) we built above.  Again, feel free to style this however you like
$template = @'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta http-equiv="X-UA-Compatible" content="ie=edge">
<style>
body {font-size: small}
img {display: none}
hr {border-top: 0.3px solid lightgrey}
</style>
</head>
<body>
...body
</body>
</html>
'@
    # Here we replace "...body" with our HTML body and save it to file
    $template -replace '...body', $body | Out-File -Force "./$file_name.html"
    $body = $null
}  # This is the end of the funtion

# Here we setup some starting variables.
$team_id = 'nnn-nnn-nnn-nnn-nnn'
$channel_id = 'nnn@thread.skype'
$channel_displayName = 'nnn.html'

# Here we call the GetMessage function. Obviously you would put this into a foreach loop and loop through each channel in your Team
GetMessages -team_id $team_id -channel_id $channel_id -file_name $channel_displayName
~~~

</div>