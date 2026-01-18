# Developer Certificate of Origin

Butler requires all contributors to sign-off on their commits, indicating agreement with the Developer Certificate of Origin (DCO).

## What is the DCO?

The DCO is a lightweight way for contributors to certify that they wrote or otherwise have the right to submit the code they are contributing. The full text of the DCO is below.

## How to Sign Off

Add `Signed-off-by` to your commit messages:

```bash
# Using -s flag
git commit -s -m "feat: add new feature"

# Or manually in commit message
git commit -m "feat: add new feature

Signed-off-by: Your Name <your.email@example.com>"
```

### Configuring Git

Make sure Git is configured with your real name and email:

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### Signing Off After the Fact

If you've already made commits without signing off:

```bash
# Amend the last commit
git commit --amend -s

# Rebase to sign multiple commits
git rebase --signoff HEAD~3  # Sign last 3 commits
```

## DCO Text

```
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.


Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

## Why DCO?

The DCO is used by many open source projects, including the Linux kernel and Kubernetes. It provides a lightweight mechanism for contributors to certify they have the right to submit their contributions, without requiring a Contributor License Agreement (CLA).

## More Information

- [DCO App](https://github.com/apps/dco) - GitHub app for DCO enforcement
- [Linux Foundation DCO](https://wiki.linuxfoundation.org/dco) - Original DCO documentation
