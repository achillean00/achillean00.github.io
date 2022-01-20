---
layout: single
title: Fixing MS Teams error 80080300 - we've run into an issue
date: '2022-01-20 10:28:03'
tags:
- "tech support"
classes: wide
typora-copy-images-to: ../images/posts/2022/${filename}/
---

When I went to open Teams this morning I was greeted by the error " We’re sorry – we’ve run into an issue". I tried a bunch of troubleshooting including:

- Signing out/back in
- Deleting everything in %appdata%\Microsoft\Teams
- Uninstalling and reinstalling Teams

none of which worked.

After more extensive googling I found a [post](https://techcommunity.microsoft.com/t5/microsoft-teams/teams-error-code-80080300-on-login/m-p/1563256) which suggested a fix:

1. Kill off Teams and make sure the tray icon has gone too
2. Find the shortcut for teams
3. Go to the compatibility tab and set the compatibility mode to *Windows 8*

On starting Team from the changed icon it worked fine. I could also then kill of Teams, undo the compatibility hack and restart Teams without issue...

