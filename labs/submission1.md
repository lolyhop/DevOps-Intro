# Lab 1 Submission

## Short Summary

Without commit signing, anyone can fake a commit by setting a name and email, for example pretending to be Yann LeCun. Git will accept it.

With commit signing, this is not possible: without the real private key, the commit cannot be verified and GitHub will mark it as Unverified.

Signing commits proves real authorship and protects commits from being modified.


## Evidence of Signed Commit

![SSH Signing Keys](img/ssh_keys.png)

The screenshot shows the SSH key added in GitHub. This is required for GitHub to verify signed commits.

![GitHub Verified Commit Badge](img/verified_commit.png)

The commit is marked as **Verified** on GitHub, confirming that it was signed with a registered SSH signing key.


## Why is commit signing important in DevOps workflows?

Commit signing is important in DevOps because:
- It prevents impersonation: no one can fake a commit using someone else’s name;
- CI/CD pipelines can trust that changes come from real contributors;
- It improves security and auditability of the Git history;
- Branch protection rules can enforce signed commits automatically;
- It helps maintain a secure and trustworthy software supply chain.
