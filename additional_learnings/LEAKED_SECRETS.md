# You've leaked a secret in your git repository - now what?
![Hagrid regretting that he spilled the secrets](img/hagrid-harry-potter.png.gif "Don't worry Hagrid, it happens to the best of us...")


Unintentionally **leaking secrets** in git repositories or any version control for that matter, is one of the most prevelant root causes for initial access of threat actors. 
According to [GitGuardian](https://www.gitguardian.com/)'s State of Secret Sprawls 2025 report[^1] almost **24 million secrets** were detected in public GitHub commits in 2024!

Examples of _secrets_ are:
- AWS Access Keys,
- API tokens, 
- a username:password combination,
- encryption keys,
- or anything you define as "_secret_" which should not be made public

In the following blog post I will explore different options of preventing leaking secrets in the first place and what to do when you were unable to prevent it.

## Prevention
> _"An ounce of prevention is worth a pound of cure"_ - Benjamin Franklin, 1736

Although Benjamin Franklin famously used that proverb to advise fire-threatend Philadelphians, it holds true in the context of IT Security hygiene. When you can prevent your secrets from becoming public you don't have to deal with the aftermath of leaked secrets. The best way to prevent publishing your secrets is to **detect** them in your code - this is called **secret detection**.

There is an abundance of commercial and free secret detection tools out there. Most CNAPPs offer secret detection capabilties. [OWASP recommends](https://owasp.org/www-community/Free_for_Open_Source_Application_Security_Tools) the following Secret Detection tools:
- [GitGuardian](https://www.gitguardian.com) - Commercial product that also offers a free version
- [gitleaks](https://github.com/gitleaks/gitleaks)
- [SAP's Credential Dig'ger](https://github.com/SAP/credential-digger)
- [Truffle Hog](https://github.com/trufflesecurity/trufflehog)
- [Yelp's detect-secrets](https://github.com/Yelp/detect-secrets)
- [Arnica](https://www.arnica.io/use-cases/hard-code-secrets)

A "good" secret detection tool should offer the following capablities:
- **Extensive Secret Type Coverage**. You don't want to have to write your own RegEx to detect secrets in your code.
- **Alerting workflow**. When your tool does detect a leaked secret you want to be notified immediately via your preferred communication method. Think e-mail, Slack messages, JIRA tickets, etc.
- **Integration and Workflow Compatibility**. It should integrate seamlessly into your Version Control Systems (VCS), CI/CD pipelines, and IDE. Wouldn't it be nice if your tool could scan for secret before you `git push` or even `git commit`? Check out the [pre-commit](https://pre-commit.com) framework.
- **Whitelisting and approval**. Sometimes you may want to intentionally allow secrets to be made public. Your tool should be able to handle such edge-cases to allow and/or ignore a secret occurence.

I've been using GitGuardian in both work-related as well as my own projects


[^1]: [The State of Secrets Sprawl 2025](https://www.gitguardian.com/state-of-secrets-sprawl-report-2025)