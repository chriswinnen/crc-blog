# You've leaked a secret in your git repository - now what?
![Hagrid regretting that he spilled the secrets](img/hagrid-harry-potter.png.gif "Don't worry Hagrid, it happens to the best of us...")


Unintentionally **leaking secrets** in git repositories or any version control for that matter, is one of the most prevelant root causes for initial access of threat actors. 
According to [GitGuardian](https://www.gitguardian.com/)'s State of Secret Sprawls 2025 report[^1] almost **24 million secrets** were detected in public GitHub commits in 2024!
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

Although Benjamin Franklin famously used that proverb to advise fire-threatend Philadelphians, it holds true in the context of IT Security hygiene. When you can prevent your secrets from becoming public you don't have to deal with the aftermath of leaked secrets. The best way to prevent publishing your secrets is to **detect** them in your code - this is called **secret detection**.

There is an abundance of commercial and free secret detection tools out there. OWASP recommends[^2] the following Secret Detection tools:
[^2]: [OWASP's free for Open Source Application Security Tools](https://owasp.org/www-community/Free_for_Open_Source_Application_Security_Tools)

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

I've been using GitGuardian in both work-related as well as my own projects and can highly recommend it.


## Reaction
![Mulan meme about API token](img/all_of_china.jpg "Once you've leaked your secret you can't put the genie back in the bottle.")

Despite all of these tools and awareness for security best practices it can still happen that you accidentally leak a secret - we're all human after all. Here's what you should do in that case.

### 0. Don't panic
Discovering an incident where you've compromised secret information by publishing it publicly can and will be stressful. However, it's crucial not to panic but instead **remain calm and take swift but planned action**.

### 1. Rotate or revoke the secret
Once you've published a secret to a public git repository you should **assume breach**, meaning that you should treat the secret as compromised no matter if it's only been online for a few minutes. Todays threat actors use sophisticated tools and techniques that allow them to move at rapid speed once they've discovered valid credentials. Chris Farris has done some research[^3] on how long it takes for public AWS Access Keys to be discovered and actively used for nefarious activities.

That's why you should immediately rotate or revoke the secret. Your approach will vary depending on the type of secret involved in the compromise. Sometimes it may be possible to "_rotate_" a secret (e.g. changing a password for user). Other options may be "_revoking_" or "_invalidating_" the secret.

The goal is to make the leaked secret **unusuable** for potential attackers.



[^3]: [Public Access Keys - 2023](https://www.chrisfarris.com/post/public-access-key-2023/)

### 2. Clean up your git history
You may be tempted to simply "_overwrite_" a leaked secret by commiting and pushing updated versions of your code to your git repository. However, Git is designed to keep an entire history of your work. And that also includes your commit history where attackers could still find an older version of your code that contains a leaked secret.

> _Leaking secrets in public repositories on GitHub and then removing them, is just like accidentally posting an embarrassing tweet, deleting it and just hoping no one saw it or took a screenshot. [^4]_
[^4]: [Exposing secrets on GitHub: What to do after leaking credentials and API keys](https://blog.gitguardian.com/leaking-secrets-on-github-what-to-do/)

Once you've revoked your secret, it cannot be used anymore. Nonetheless, exposing secrets in your version control looks unprofessional and might raise concerns (even when they are expired). That's why you should consider **rewriting your git history**.


I recently ~~wanted~~ had to rewrite my git history because I had hardcoded (_bad..._) and pushed (_worse..._) the `api_token` of my CloudFlare terraform provider configuration in my `providers.tf`.

I used `git-filter-repo`[^5] for that purpose. Here are the steps I took to replace `api_token = "abcd-1234-efgh-56789"` with `api_token = "REMOVED_FROM_GIT_HISTORY"` in all commits of the `providers.tf` file in the entire git history. 

If you find yourself in a similar position you can follow these steps:

#### Step 0: Install `git-filter-repo`
...if you haven't already, follow the instructions in the official [installation guide](https://github.com/newren/git-filter-repo/blob/main/INSTALL.md).

#### Step 1: Backup you repository 
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
git remote rm origin
```

You'll add it back later.

#### Step 4: Constructing the `git-filter-repo` command
Let's say you want to replace a specific string (`find`) with another string (`replace`) from a specific file (`file_name`) in all commits:

```bash
git filter-repo --path file_name --replace-text <(echo 'find==>replace')
```

Let's break down this command:
- `git filter-repo`: The command to invoke the tool.
- `--path providers.tf`: This tells `git-filter-repo` to focus its operation only on the `file_name` file. This is crucial because it makes the operation faster and safer, as it won't touch other files.
- `--replace-text`: This is the core action. It instructs it to perform text replacement. It expects a file (or standard input) where each line defines a `find==>replace` pattern.
- `<(echo 'find==>replace')`: In essence, this dynamically creates a temporary "file" containing the single line `find==>replace` and passes its path to the `--replace-text` option. This avoids having to create a separate text file (`replace_rules.txt`) just for this one rule.

For my example I used the following command to replace the API token (`abcd-1234-efgh-56789`) with a redacted version (`REMOVED_FROM_GIT_HISTORY`) in the `providers.tf` file in my entire git history:
```bash
git filter-repo --path providers.tf --replace-text <(echo 'abcd-1234-efgh-56789==>REMOVED_FROM_GIT_HISTORY')
```

#### Step 5: Re-add your remote origin
After you've ran the command, your local history is rewritten. You need to re-establish the connection to your remote repository:

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


[^5]: Check out the [git-repo-filter manual](https://htmlpreview.github.io/?https://github.com/newren/git-filter-repo/blob/docs/html/git-filter-repo.html) for more details.

<!-- GitGuardian has an excellent blog post and cheatsheet[^5] that explains the steps in more details. -->

<!-- [^6]: [Rewriting your git history, removing files permanently](https://blog.gitguardian.com/rewriting-git-history-cheatsheet/) -->