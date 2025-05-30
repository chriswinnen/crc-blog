# You've leaked a secret in your git repository - now what?
![Hagrid regretting that he spilled the secrets](img/hagrid-harry-potter.png.gif "Don't worry Hagrid, it happens to the best of us...")


Unintentional **secrets leaks** in git repositories or any version control for that matter, are one of the most prevalent root causes for initial access of threat actors. 
According to the [GitGuardian](https://www.gitguardian.com/)'s State of Secret Sprawls 2025 report[^1], GitGuardian detected almost **24 million secrets** in public GitHub commits in 2024!
[^1]: [The State of Secrets Sprawl 2025](https://www.gitguardian.com/state-of-secrets-sprawl-report-2025)

Examples of _secrets_ are:
- AWS Access Keys,
- API tokens, 
- a username:password combination,
- encryption keys,
- or anything you define as "_secret_" which should not be made public

In the following blog post I will explore different options of preventing leaking secrets in the first place and what to do when you were unable to prevent it.

## Prevention
> _"An ounce of prevention is worth a pound of cure"_ - Benjamin Franklin, 1736

Although Benjamin Franklin famously used that proverb to advise fire-threatened Philadelphians, it holds true in the context of IT Security hygiene. When you can prevent your secrets from becoming public you don't have to deal with the aftermath of leaked secrets. The best way to prevent publishing your secrets is to **detect** them in your code - this is called **secret detection**.

There is an abundance of commercial and free secret detection tools out there. OWASP recommends[^2] the following Secret Detection tools:
[^2]: [OWASP's free for Open Source Application Security Tools](https://owasp.org/www-community/Free_for_Open_Source_Application_Security_Tools)

- [GitGuardian](https://www.gitguardian.com) - Commercial product that also offers a free version
- [gitleaks](https://github.com/gitleaks/gitleaks)
- [SAP's Credential Dig'ger](https://github.com/SAP/credential-digger)
- [Truffle Hog](https://github.com/trufflesecurity/trufflehog)
- [Yelp's detect-secrets](https://github.com/Yelp/detect-secrets)
- [Arnica](https://www.arnica.io/use-cases/hard-code-secrets)

A "good" secret detection tool should offer the following capabilities:
- **Extensive Secret Type Coverage**. Look for tools that spare you the effort of writing custom RegEx for secret detection.
- **Alerting workflow**. When your tool does detect a leaked secret you want to be notified immediately via your preferred communication method. Think e-mail, Slack messages, JIRA tickets, etc.
- **Integration and Workflow Compatibility**. It should integrate seamlessly into your Version Control Systems (VCS), CI/CD pipelines, and IDE. Wouldn't it be nice if your tool could scan for secrets before you `git push` or even `git commit`? Check out the [pre-commit](https://pre-commit.com) framework.
- **Whitelisting and approval**. Sometimes you may want to intentionally allow secrets to be made public. Your tool should be able to handle such edge-cases to allow and/or ignore a secret occurrence.

I've been using GitGuardian in both work-related as well as my own projects and can highly recommend it.


## Reaction
![Mulan meme about API token](img/all_of_china.jpg "Once you've leaked your secret you can't put the genie back in the bottle.")

Despite all of these tools and awareness for security best practices it can still happen that you accidentally leak a secret - we're all human after all. Here's what you should do in that case.

### 0. Don't panic
Discovering an incident where you've compromised secret information by publishing it publicly can and will be stressful. However, it's crucial not to panic but instead **remain calm and take swift but planned action**.

### 1. Rotate or revoke the secret
Once you've published a secret to a public git repository you should **assume breach**, meaning that you should treat the secret as compromised no matter if it's only been online for a few minutes. Today's threat actors use sophisticated tools and techniques that allow them to move at rapid speed once they've discovered valid credentials. Chris Farris has done some research[^3] on how long it takes for public AWS Access Keys to be discovered and actively used for nefarious activities.

That's why you should immediately rotate or revoke the secret. Your approach will vary depending on the type of secret involved in the compromise. Sometimes it's possible to "_rotate_" a secret (e.g. changing a password for a user). Other times you can only "_revoke_", "_delete_", or "_invalidate_" a secret.

The goal is to make the leaked secret **unusable** for potential attackers.



[^3]: [Public Access Keys - 2023](https://www.chrisfarris.com/post/public-access-key-2023/)

### Clean up your git history (optional)
> _Leaking secrets in public repositories on GitHub and then removing them, is just like accidentally posting an embarrassing tweet, deleting it and just hoping no one saw it or took a screenshot. [^4]_
[^4]: [Exposing secrets on GitHub: What to do after leaking credentials and API keys](https://blog.gitguardian.com/leaking-secrets-on-github-what-to-do/)

You may be tempted to simply "_overwrite_" a leaked secret by committing and pushing updated versions of your code to your git repository. However, Git is designed to keep an entire history of your work. And that also includes your commit history where attackers could still find an older version of your code that contains a leaked secret.

Once you've revoked your secret, it cannot be used anymore. Nonetheless, exposing secrets in your version control looks unprofessional and might raise concerns (even when they are expired). That's why you should consider **rewriting your git history**.

One tool to efficiently rewrite your git history is `git-repo-filter`. It is also what GitHub recommends [^5] when removing sensitive data from a repository. [^5]: [Removing sensitive data from a repository](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)

I've listed the steps to rewrite your git history below.

#### Step 0: Install `git-filter-repo`
If you haven't already, you can follow the instructions in the official [installation guide](https://github.com/newren/git-filter-repo/blob/main/INSTALL.md).

#### Step 1: Backup your repository 
Seriously, copy your entire Git repository to another location. If something goes wrong, you can always revert to this backup. 

#### Step 2: Ensure a clean working directory
`git-filter-repo` requires a clean working directory and no uncommitted changes.

```bash
git status
```

If you have anything uncommitted, stash it (`git stash`) or commit it.

#### Step 3: Remove all remote origins (temporarily)
This prevents accidental pushing of the rewritten history to your remote before you're ready.

```bash
git remote remove origin
```

You'll add it back later.

#### Step 4: Constructing the `git-filter-repo` command
Let's say you want to replace a specific string (`find`) with another string (`replace`) from a specific file (`file_name`) in all commits:

```bash
git filter-repo --path file_name --replace-text <(echo 'find==>replace')
```

Let's break down this command[^6]:
- `git filter-repo`: The command to invoke the tool.
- `--path file_name`: This tells `git-filter-repo` to focus its operation only on the `file_name` file. This is crucial because it makes the operation faster and safer, as it won't touch other files.
- `--replace-text`: This is the core action. It instructs it to perform text replacement. It expects a file (or standard input) where each line defines a `find==>replace` pattern.
- `<(echo 'find==>replace')`: In essence, this dynamically creates a temporary "file" containing the single line `find==>replace` and passes its path to the `--replace-text` option. This avoids having to create a separate text file (`replace_rules.txt`) just for this one rule.

[^6]: Check out the [git-repo-filter manual](https://htmlpreview.github.io/?https://github.com/newren/git-filter-repo/blob/docs/html/git-filter-repo.html) for more details.

#### Step 5: Re-add your remote origin
After you've run the command, your local history is rewritten. You need to re-establish the connection to your remote repository:

```bash
git remote add origin https://<URL_TO_YOUR_REMOTE_GIT>
```

### Step 6: Force Push the rewritten history
> [!CAUTION]
> Rewriting Git history is a powerful operation that fundamentally alters the project's commit log, leading to **disrupted workflows for collaborators who will need to re-clone or force update their repositories, and potential data loss if not executed carefully.** Inform any collaborators before performing this step, as they will need to re-clone or perform complex Git operations on their end. It can also complicate auditing and make it harder to trace the true lineage of changes, as commit hashes will change.

This is the most critical step and requires overwriting the remote history.

```bash
git push --force-with-lease --set-upstream origin main
```

- `--force-with-lease`: This is a safer form of force push. It will only push if the remote branch has not changed since you last fetched it. This prevents overwriting changes made by someone else in the interim.
- `--set-upstream origin main`: This tells Git that your local `main` branch should track the `main` branch on the `origin` remote.

#### Step 7: Verify the changes
Finally, confirm that the sensitive information has been successfully removed from your repository's history.

- Online Verification: Go to your GitHub repository on the web, navigate to the file that contained the secret, and click on "History" or "Blame." You should see the line replaced in all commits.
- New Clone Test: Clone your repository into a fresh, empty directory and inspect older commits (e.g., `git log --oneline`, then `git show <old-commit-sha>:providers.tf`) to ensure the token is no longer present anywhere in the history.

### Monitor your logs for suspicious activities
As with any security incident, it's important to monitor for suspicious activities. Attackers frequently leverage compromised credentials to gain unauthorized access, move laterally within your systems, or exfiltrate sensitive data. By scrutinizing authentication logs, API activity, and access patterns for anomalies – such as logins from unusual locations, unexpected API calls, or resource creation outside of normal operations – you can quickly identify and respond to any post-compromise activity, limiting potential damage and preventing further breaches.




## Personal experience
> _"By three methods we may learn wisdom: First, by reflection, which is noblest; Second, by imitation, which is easiest; and third by experience, which is the bitterest."_ - Confucius

I've recently chosen the bitterest option and learned wisdom by experience.

I had hardcoded (_bad..._) the `api_token` of my CloudFlare terraform provider configuration in my `providers.tf` and then pushed it to a public GitHub repository (_worse..._). Here's what I did in response to that:

### I revoked my CloudFlare API Token
Luckily, Cloudflare offers the option to _roll_ an existing API token. This essentially **invalidates** the "old" API token. It also has the benefit of keeping the existing permission scopes. 

CloudFlare offers an [API](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/methods/verify/) to verify whether a token works. After invalidating my leaked token I made sure that it would be unusable for anyone who may have discovered it:

```bash
curl "https://api.cloudflare.com/client/v4/accounts/<MY_ACCOUNT_ID>/tokens/verify" \
     -H "Authorization: Bearer abcd-1234-efgh-56789"
```

The response I got validated that the API Token is no longer working:
```json
{
  "success": false,
  "errors": [
    {
      "code": 1000,
      "message": "Invalid API Token"
    }
  ],
  "messages": [],
  "result": null
}
```

I could have stopped here, since the published API token was now rendered useless. However, I decided to go the extra mile and apply my lessons-learnt.

### I changed my git history
As described in the previous section, I used `git-filter-repo` to replace my hardcoded API token throughout my entire git history.

For my example I used the following command to replace the API token (`abcd-1234-efgh-56789`) with a redacted version (`REMOVED_FROM_GIT_HISTORY`) in the `providers.tf` file in my entire git history:
```bash
git filter-repo --path providers.tf --replace-text <(echo 'abcd-1234-efgh-56789==>REMOVED_FROM_GIT_HISTORY')
```

After performing all the necessary steps I was able to verify that my commit history no longer showed the hardcoded API token, but a redacted value instead:

![The old commits no longer show a hardcoded API token.](img/rewritten_git_history.png "It is possible to rewrite history - with a bit of extra work.")

### I set up ggshield, pre-commit & pre-push secret scanning, and installed the GitGuardian Visual Studio Code extension
To avoid accidentally leaking secrets in the future, I went ahead and implemented what's called "Shift Left Security".

> _Shift-left security is the practice of performing code and software security assurance processes as early as possible in the software development lifecycle (SDLC)._ - definition from the Wiz Academy[^7]
[^7]: [Shift Left Explained: What It Means to Shift Security Left](https://www.wiz.io/academy/shift-left-security)


### Setting up ggshield locally
ggshield[^8] is the command line tool for the GitGuardian secret detection engine. You can use it to search for secrets in files, repositories, Docker images and more.
[^8]: [ggshield documentation](https://www.gitguardian.com/ggshield)


Below is an example of scanning a specific file (`providers.tf`) for secrets:

```bash
ggshield secret scan path providers.tf                                       
Scanning... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 1 / 1

> providers.tf: 1 secret detected

>> Secret detected: Generic High Entropy Secret
   Validity: No Checker
   Occurrences: 1
   Known by GitGuardian dashboard: NO
   Incident URL: N/A
   Secret SHA: 3e56b8b21a7e2846a37c797a5115918092442c161e47c067dc8894dd29868e58
   Detector documentation: https://docs.gitguardian.com/secrets-detection/secrets-detection-engine/detectors/generics/generic_high_entropy_secret

12 | # Hardcoded credentials for initial test
13 | provider "cloudflare" {
14 |   api_token = "abcd*********************-****6789"
                    |_____________apikey_____________|
15 | }
```

This shows that `ggshield` was able to detect my hardcoded API token and alert me on it. That's a good start! However, having to manually run the secret detection tool is not a great "Developer Experience".

> [!TIP]
> Running `ggshield` will create a local cache of the detected secrets (`.cache_ggshield`). You may want to add this to your `.gitignore` to prevent it from being accidentally committed to your version control system.

### Setting up pre-commit and pre-push scans
GitGuardian offers a way to automatically scan your code for secrets **before** your commits and/or pushes. That means you don't have to explicitly invoke `ggshield` manually to find the secrets in your code.

#### Pre-commit hook
A pre-commit hook is a client-side git hook that runs right before the commit is created. 

To enable pre-commit scanning, run:
```bash
ggshield install --mode global
```

GitGuardian will now warn and prevent you from commiting code that contains secrets. In the example below, I tried to commit the `providers.tf` file which contained the hardcoded API Token:
```bash
git commit -m"Hardcoded the api_token"
Scanning... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 1 / 1

> commit://staged/providers.tf: 1 secret detected

>> Secret detected: Generic High Entropy Secret
   Validity: No Checker
   Occurrences: 1
   Known by GitGuardian dashboard: NO
   Incident URL: N/A
   Secret SHA: 3e56b8b21a7e2846a37c797a5115918092442c161e47c067dc8894dd29868e58
   Secret in Secrets Manager: False
   Detector documentation: https://docs.gitguardian.com/secrets-detection/secrets-detection-engine/detectors/generics/generic_high_entropy_secret

      | \ No newline at end of file
   12 | +# Hardcoded credentials for initial test
   13 | +provider "cloudflare" {
   14 | +  api_token = "abcd*********************-****6789"
                        |_____________apikey_____________|
   15 | +}

> How to remediate

  Since the secret was detected before the commit was made:
  1. replace the secret with its reference (e.g. environment variable).
  2. commit again.

> [Apply with caution] If you want to bypass ggshield (false positive or other reason), run:
  - if you use the pre-commit framework:

    SKIP=ggshield git commit -m "<your message>"
```

Let's examine this in more detail:
- GitGuardian **rejected** my commit because it found a secret
- It **highlighted** where the secret was found in my code (`Line 14`)
- It **suggested remediation steps** (`replce the secret with environment variable`)
- It showed how to **ignore** the secret finding (by adding`SKIP=ggshield` before the `git commit` command)


#### Pre-push hook
A pre-push hook is a client-side git hook that runs right before a reference is pushed to a remote (git push). 

To enable pre-push scanning, run:
```bash
ggshield install --mode global -t pre-push
```

GitGuardian will now warn and prevent you from pushing code that contains secrets. In the example below, I tried to push the `providers.tf` file which contained the hardcoded API Token:

```bash
git push
Scanning... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 1 / 1

commit d6803caac9883af4ab4108fe037553313afa3922
Date: Fri May 30 10:53:00 2025 +1000

> commit://d6803caac9883af4ab4108fe037553313afa3922/providers.tf: 1 secret detected

>> Secret detected: Generic High Entropy Secret
   Validity: No Checker
   Occurrences: 1
   Known by GitGuardian dashboard: NO
   Incident URL: N/A
   Secret SHA: 3e56b8b21a7e2846a37c797a5115918092442c161e47c067dc8894dd29868e58
   Secret in Secrets Manager: False
   Detector documentation: https://docs.gitguardian.com/secrets-detection/secrets-detection-engine/detectors/generics/generic_high_entropy_secret

      | \ No newline at end of file
   12 | +# Hardcoded credentials for initial test
   13 | +provider "cloudflare" {
   14 | +  api_token = "abcd*********************-****6789"
                        |_____________apikey_____________|
   15 | +}

> How to remediate

  Since the secret was detected before the push BUT after the commit, you need to:
  1. rewrite the git history making sure to replace the secret with its reference (e.g. environment variable).
  2. push again.

  To prevent having to rewrite git history in the future, setup ggshield as a pre-commit hook:
    https://docs.gitguardian.com/ggshield-docs/integrations/git-hooks/pre-commit

> [Apply with caution] If you want to bypass ggshield (false positive or other reason), run:
  - if you use the pre-commit framework:

    SKIP=ggshield-push git push
```

Let's examine this in more detail:
- GitGuardian **rejected** my push because it found a secret.
- It **highlighted** where the secret was found in my code (in commit `d6803caac9883af4ab4108fe037553313afa3922` on `Line 14`).
- It **suggested remediation steps** (`replce the secret with environment variable`).
- It showed how to **ignore** the secret finding (by adding`SKIP=ggshield` before the `git push` command).

### Setting up the Visual Studio Code integration
GitGuardian also offers an integration for Visual Studio Code (VSC)[^9].
[^9]: You can install it from the [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=gitguardian-secret-security.gitguardian)

Once you've installed the VSC integration, GitGuardian will highlight it in your editor:

![Example of a highlighted secret that was detected](img/incident_highlighted_dark.png "GitGuardian will highlight all the secrets it finds in your code.")

## The easiest method of learning wisdom
My recent experience was a bitter lesson, but one that ultimately strengthened my security posture and reinforced the importance of proactive defense. 

You don't have to learn this wisdom the "bitter" way yourself. You can use Confucius's easiest method of learning wisdom (_"imitation"_) by taking action today:

Integrate a secret detection tool like GitGuardian into your workflow and set up those crucial pre-commit and pre-push hooks. It’s a small effort that yields monumental security benefits, ensuring your secrets stay secret.