---
title: Understanding client and server throttling in MSAL.NET
---

# Understanding client and server throttling in MSAL.NET

## Server throttling

Azure AD throttles applications when you call it too often. Most often this happens when token caching is not used, for example, because:

1. Token caching is not setup correctly (see [Token cache serialization](/azure/active-directory/develop/msal-net-token-cache-serialization)).
2. Not calling `AcquireTokenSilent` before calling `AcquireTokenInteractive`, `AcquireTokenByUsernamePassword` (see [AcquireTokenSilent](https://github.com/AzureAD/microsoft-authentication-library-for-dotnet/wiki/AcquireTokenSilentAsync-using-a-cached-token)).
3. If you are asking for a scope which does not make sense for MSA users like `User.ReadBasic.All`, which causes cache misses.

The server signals throttling in 2 ways:

- for `client_credentials` grant, i.e. `AcquireTokenForClient`, Azure AD will reply with `429 Too Many Requests` with a `Retry-After: 60` header.
- for user facing calls, Azure AD will send an message which results in an `MsalUiRequiredException` with error code `invalid_grant` and a message `AADSTS50196: The server terminated an operation because it encountered a loop while processing a request`.

## Client throttling

MSAL detects certain conditions (see below) where the application should not make repeated calls to Azure AD. If a call is made, then an `MsalThrottledServiceException` or an `MsalThrottledUiRequiredException` is thrown by MSAL. These are subtypes of `MsalServiceException`, so this behavior does not introduce a breaking change. If MSAL would not apply client-side throttling, the application would still not be able to acquire tokens, as Azure AD would throw the error.

## Conditions to get throttled

### Azure AD is telling the application to back off

If the server is having problems or if an application is requesting tokens too often, Azure AD will respond with `HTTP 429 (Too Many Requests)` and with `Retry-After` header, `Retry-After X seconds`. The application will see an `MsalServiceException` with [header details](https://github.com/AzureAD/microsoft-authentication-library-for-dotnet/wiki/retry-after). The throttling state is maintained for the X seconds. Affects all flows. Introduced in 4.13.0.

The most likely culprit is that you have not setup token caching. See [Token cache serialization in MSAL.NET](/azure/active-directory/develop/msal-net-token-cache-serialization).

### Azure AD is having problems

If Azure AD is having problems it may respond with an HTTP 5xx error code with no `Retry-After` header. The throttling state is maintained for 1 minute. Affects only public client flows. Introduced in 4.13.0

### Application is ignoring `MsalUiRequiredException`

MSAL throws `MsalUiRequiredException` when authentication cannot be resolved silently and the end-user needs to use a browser. This is a common occurrence when a tenant admin introduced Multi-Factor Authentication or when a user's password expires. Retrying the silent authentication cannot succeed. The throttling state is maintained for 2 minutes. Affects only the `AcquireTokenSilent`. Introduced in 4.14.0