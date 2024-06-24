---
sidebar_position: 1
---

# Configure multiple SSH for github

If you wanna configure SSH for different accounts, please read follow contents:

## 1. Create your first React Page

Generate 2 pairs of ssh-keys:

```jsx title="src/pages/my-react-page.js"
ssh-keygen -t rsa -f "customized name"
```

Then you get id_rsa and id_rsa.pub

## 2. Configure ssh in your github

Add each of  ~.pub file in your github settings

## 3. Create or update your config file in .ssh folder

Create a config file if you don't have it in .ssh folder

```bash
# key1 one github account
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/id-rsa

# key2 another github account
Host me.github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/jiaxi

```

## 4. Change the remote address of the corresponding warehouse to a customized name

Create a config file if you don't have it in .ssh folder

```bash
PS D:\personalCode\jiaxi.com> git remote -v
origin  git@me.github.com:Lingerssss/jiaxi.com.git (fetch)
origin  git@me.github.com:Lingerssss/jiaxi.com.git (push)
```

