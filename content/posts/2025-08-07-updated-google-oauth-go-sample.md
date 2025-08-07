+++
author = "hannibal"
categories = ["golang", "oauth", "authentication"]
date = "2025-08-07T01:01:00+01:00"
title = "Updated Google OAuth Go Sample - Modern Authentication with Improved UI"
url = "/2025/08/07/updated-google-oauth-go-sample"
comments = true
+++

# Updated Google OAuth Go Sample

Hello.

I've recently updated my Google OAuth Go sample application[^1] to follow modern standards and practices.

The update brings several improvements including a cleaner UI, better error handling, and more robust authentication flows.

## What's New

New interface for the login side:

![Welcome Screen](/img/2025/08/welcome.png)

Nicer errors:

![Error Screen](/img/2025/08/error.png)

Welcome message:

![Logged In Screen](/img/2025/08/logged-in.png)

Battle arena:

![Battle Arena](/img/2025/08/battle-arena.png)

And lastly, logout:

![Logout Screen](/img/2025/08/logout.png)

## Technical Updates

I added OAuth 2.0 best practices and some proper flows with auth middleware.

Setup remains simple - obtain credentials from Google Cloud Console, configure the redirect URI to `http://127.0.0.1:9090/auth`, and place your `creds.json` in the project root.

It runs under 9090 and provides a protected `/battle/field` route to showcase authenticated access.

## Conclusion

That's about it.

Thanks for reading!

[^1]: https://github.com/Skarlso/google-oauth-go-sample