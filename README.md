# Creating a bookmarklet to share links to AWS Console pages

## TL;DR

[Drag me to your Bookmark Bar][1]

## Details

AWS recommends using multiple accounts to separate different environments, such
as production, staging, and development. This is a good practice, but it is
cumbersome as you need  to switch between accounts frequently to manage and
debug resources.

A solution to this problem is using IAM Identity Center, which allows you  to
easily switch between accounts.

One problem  I've always had with AWS however, is that  it is not possible
to share links to AWS Console pages with colleagues. The AWS console page URL
does not contain the account ID, so the link is not valid for other users unless
they happen to be logged in to the same account.

E.g. the link
[https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#SnapshotDetails:snapshotId=snap-09e338b471109c7ac](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#SnapshotDetails:snapshotId=snap-09e338b471109c7ac)
links to a snapshot that I own. But nothing in the URL tells you what account it is in, and thus
AWS redirects you to the generic  "Sign in to Console" page prompting for your Root IAM user credentials
or even worse it navigates to that page in a different account and says "Resource doesn't exist"

This is unlike GCP; where everything is in a single "Account", but things are segregated in "Projects". and the URL contains the project id.
This means when you share a link to the GCP console, people will be able to get exactly where you were.


## An improvement just landed

AWS Just announced the [Create shortcut button](https://docs.aws.amazon.com/singlesignon/latest/userguide/createshortcutlink.html?icmpid=docs_sso_console)
on the AWS IAM Identity Center Start Page.

<img width="513" alt="A picture showing a pop-up on AWS with the text: We've added a Create shortcut button so you can generate secure shortcut links to AWS Management Console pages that you can bookmark or share with others that have AWS account access. Learn more. The Create Shortcut button is highlighted by the author." src="https://gist.github.com/assets/628387/7606c307-16c8-48c0-a679-202fc2cf160d">

You can then paste your Console URL into the screen and AWS will create a Shortcut that will include the account id and will redirect you
to AWS IAM Identity Center to sign in to the specific account automatically!
<img width="884" alt="image" src="https://gist.github.com/assets/628387/7ca6bb73-140b-419b-9815-c728e542f81e">

E.g. it generated the URL [https://nixos.awsapps.com/start/#/console?account_id=427812963091&destination=https%3A%2F%2Fus-east-1.console.aws.amazon.com%2Fec2%2Fhome%3Fregion%3Dus-east-1%23SnapshotDetails%3AsnapshotId%3Dsnap-09e338b471109c7ac](https://nixos.awsapps.com/start/#/console?account_id=427812963091&destination=https%3A%2F%2Fus-east-1.console.aws.amazon.com%2Fec2%2Fhome%3Fregion%3Dus-east-1%23SnapshotDetails%3AsnapshotId%3Dsnap-09e338b471109c7ac)
after me pasting in [https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#SnapshotDetails:snapshotId=snap-09e338b471109c7ac](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#SnapshotDetails:snapshotId=snap-09e338b471109c7ac).

The new URL will prompt you to log in to my AWS account through SSO, and afterwards redirect you to the specific resource! Amazing!


https://gist.github.com/assets/628387/64c5fc65-ff27-4531-8fdb-23b2235a7e46

## This is almost perfect ... but a button would be nicer

I need to go to the AWS IAM Identity Center portal, copy in the URL I just visited, and then copy out the URL to share with colleageus. 
This is cumbersome!

I wish AWS just adds a button "Create Shortlink" to every AWS Console page. But they don't have that yet so we have to hack something together ourselves!
Enter: bookmarklets.

In order to form a shortlink we need a few components:

* the AWS Identity Center Portal URL
* The target Account ID
* The destination page we want to navigate to

Getting the destination is easy. `window.top.location.href` will do.
But how do we get the Portal URL and the account ID?
Looking through the JS source code of the Console I find that Telemetry code in AWS extracts
these bits of information from the cookie. And there is a cookie key named `aws-userInfo` that contains exactly the info we need!
```javascript
  function parseCookie(e, t = document.cookie) {
    const n = t.match(`${e}=([^;]*)`);
    if (n && n[1]) return n[1];
  }
  /* undocumented but this  is  how AWS  telemetry gets user info */
  const sessionData = JSON.parse(
    decodeURIComponent(parseCookie("aws-userInfo"))
  );
  console.log(sessionData);
```

```json
{
    "arn": "arn:aws:sts::427812963091:assumed-role/AWSReservedSSO_AWSAdministratorAccess_392b3f25267f6536/arianvp",
    "alias": "427812963091",
    "username": "assumed-role%2FAWSReservedSSO_AWSAdministratorAccess_392b3f25267f6536%2Farianvp",
    "keybase": " XXXX",
    "issuer": "https://nixos.awsapps.com/start/#/console?account_id=427812963091&role_name=AWSAdministratorAccess&destination=https%3A%2F%2Fconsole.aws.amazon.com",
    "signinType": "PUBLIC"
}
```

Putting all the bits together we can construct a share URL to the current open page with some JS:
```javascript
(function () {
  function parseCookie(e, t = document.cookie) {
    const n = t.match(`${e}=([^;]*)`);
    if (n && n[1]) return n[1];
  }
  /* undocumented but this  is  how AWS  telemetry gets user info */
  const sessionData = JSON.parse(
    decodeURIComponent(parseCookie("aws-userInfo"))
  );
  const issuer = new URL(sessionData.issuer);
  const hash = issuer.hash;
  const [front, params] = hash.split("?");
  const urlParams = new URLSearchParams(params);
  urlParams.set("destination", window.top.location.href);
  const newHash = `${front}?${urlParams.toString()}`;
  issuer.hash = newHash;
  const el = document.activeElement;
  navigator.clipboard.writeText(issuer.href);
  alert(`Copied ShortLink ${issuer.href} to clipboard`);
  el.focus();
})();
```

Encoding that into a URL gives us:

```
console.log(`javascript:${encodeURIComponent(`(function () {
  function parseCookie(e, t = document.cookie) {
    const n = t.match(\`\${e}=([^;]*)\`);
    if (n && n[1]) return n[1];
  }
  /* undocumented but this  is  how AWS  telemetry gets user info */
  const sessionData = JSON.parse(
    decodeURIComponent(parseCookie("aws-userInfo"))
  );
  const issuer = new URL(sessionData.issuer);
  const hash = issuer.hash;
  const [front, params] = hash.split("?");
  const urlParams = new URLSearchParams(params);
  urlParams.set("destination", window.top.location.href);
  const newHash = \`\${front}?\${urlParams.toString()}\`;
  issuer.hash = newHash;
  const el = document.activeElement;
  navigator.clipboard.writeText(issuer.href);
  alert(\`Copied ShortLink \${issuer.href} to clipboard\`);
  el.focus();
})();`)}`);

```

And we turn that into a bookmarklet URL.

[1]: javascript:(function%20()%20%7B%0A%20%20function%20parseCookie(e%2C%20t%20%3D%20document.cookie)%20%7B%0A%20%20%20%20const%20n%20%3D%20t.match(%60%24%7Be%7D%3D(%5B%5E%3B%5D*)%60)%3B%0A%20%20%20%20if%20(n%20%26%26%20n%5B1%5D)%20return%20n%5B1%5D%3B%0A%20%20%7D%0A%20%20%2F*%20undocumented%20but%20this%20%20is%20%20how%20AWS%20%20telemetry%20gets%20user%20info%20*%2F%0A%20%20const%20sessionData%20%3D%20JSON.parse(%0A%20%20%20%20decodeURIComponent(parseCookie(%22aws-userInfo%22))%0A%20%20)%3B%0A%20%20const%20issuer%20%3D%20new%20URL(sessionData.issuer)%3B%0A%20%20const%20hash%20%3D%20issuer.hash%3B%0A%20%20const%20%5Bfront%2C%20params%5D%20%3D%20hash.split(%22%3F%22)%3B%0A%20%20const%20urlParams%20%3D%20new%20URLSearchParams(params)%3B%0A%20%20urlParams.set(%22destination%22%2C%20window.top.location.href)%3B%0A%20%20const%20newHash%20%3D%20%60%24%7Bfront%7D%3F%24%7BurlParams.toString()%7D%60%3B%0A%20%20issuer.hash%20%3D%20newHash%3B%0A%20%20const%20el%20%3D%20document.activeElement%3B%0A%20%20navigator.clipboard.writeText(issuer.href)%3B%0A%20%20alert(%60Copied%20ShortLink%20%24%7Bissuer.href%7D%20to%20clipboard%60)%3B%0A%20%20el.focus()%3B%0A%7D)()%3B

