# 两种方法给不同github账号配置SSH

SSH 能够自动加密和解密 SSH 客户端与服务端之间的网络数据。以github为例，只需生成公钥与私钥对，然后将公钥配置到账户中便可直接与github通信。

然而有时可能需要需要管理不同账号或网站下的仓库，此时因为一个公钥不能被重复使用我们就需要为每个账号单独配置SSH。以下有两种方式仅供参考。



## 第一种方式：通过配置文件管理

### (1) 生成密钥对

首先自然需要生成两组密钥对，-f 参数后加一个自定的名字以作区分。否则会默认覆盖.ssh中的密钥对。

```jsx title="src/pages/my-react-page.js"
ssh-keygen -t rsa -f "jiaxi(customized name)"
```

### (2) 将公钥配置到github账户中

在github/setiings/SSH and GPG keys中即可添加公钥


### (3) 为本机SSH创建一个配置文件

这里需要至少修改一组Host，IdentityFile后面指定本地存放的私钥

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



### (4) 更新git remote地址

在第三步中我们将key2的Host改为了me.github.com，所以需要将使用这个SSH的仓库的地址更新

```bash
PS D:\personalCode\jiaxi.com> git remote -v
origin  git@me.github.com:Lingerssss/jiaxi.com.git (fetch)
origin  git@me.github.com:Lingerssss/jiaxi.com.git (push)
```



## 第二种方式：生成不同类型的密钥对

在生成密钥对时，我们通过参数-t 选择生成不同类型的密钥。如rsa, dsa, ecdsa

```
ssh-keygen [-t dsa | ecdsa | ecdsa-sk | ed25519 | ed25519-sk | rsa]
```

当我们为两个不同账户配置不同类型的key时，无需添加配置文件即可正常工作。

这个方式明显的缺点是已经使用过的类型无法再使用。

## 