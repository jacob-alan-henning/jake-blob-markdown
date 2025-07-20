# You're not afraid of workload federation are you?

You're copying creds into GitHub secrets, aren't you Morgan?

![suprise](images/doakes.jpeg)

Its easier than you think; despite the best efforts of your cloud providers documentation. 

two steps

* create an identity in your cloud provider
* create a trust relationship with that identity and the repo 

OIDC can be complicated but with a trusted provider where you don't have to think about PKI its brain dead. 

In aws you create an identity provider - you need the oidc provider and audience url. Like 5 clicks in the iam console or 10 lines of terraform

then you add a trust policy to a role like this
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<arn>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": [
                        "repo:jacob-alan-henning/jake-blob-markdown:ref:refs/heads/main"
                    ]
                }
            }
        }
    ]
}
```
Then a 4 line change to your workflow you're good. 

Azure and GCP are again basically the same 

So stop authenticating to your cloud provider with secrets in your pipeline. It won't expire and the security people seem to think its better. 
