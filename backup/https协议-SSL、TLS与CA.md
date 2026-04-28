<img width="714" height="695" alt="Image" src="https://github.com/user-attachments/assets/38c914a9-ebc6-4347-8890-cde209cfa49b" />

## 基本概念
- SSL（Secure Sockets Layer，安全套接字层）：
  一种用于保护网络通信安全的加密协议，通过加密数据（如网页、邮件）防止窃听和篡改。SSL 是 TLS 的前身，现已被 TLS 取代，但仍常被统称为“SSL/TLS”。
- TLS（Transport Layer Security，传输层安全）：
  SSL 的升级版，是目前用于 HTTPS 等安全通信的标准协议。它通过公钥/私钥加密和会话密钥，确保客户端与服务器之间的数据安全、完整和身份验证。
- CA（Certificate Authority，证书颁发机构）：
  可信的第三方机构，负责颁发和验证 数字证书，证明服务器的公钥和身份真实可信。CA 用自己的私钥为证书签名，客户端用 CA 的公钥验证证书（如 Let’s Encrypt、DigiCert）。

联系：在 HTTPS 中，TLS（或旧的 SSL）通过握手传递服务器的公钥（在数字证书中），CA 保证证书的真实性，确保通信安全。

## 什么是https协议
- https协议：http+加密+认证+完整性保护。
- 把添加了加密及认证机制的 http 称为 https（HTTP Secure）。
- https协议并非应用层一种新的协议。只是 HTTP 通信接口部分用 SSL（Secure Socket Layer） 和 TLS（Transport Layer Security） 协议代替而已。

<img width="597" height="294" alt="Image" src="https://github.com/user-attachments/assets/62cb473e-c2db-4cb4-aff2-3b1555099586" />

## 两种加密技术
- 公开密钥加密（非对称加密）：使用公钥加密，私钥解密。或者私钥签名，公钥验证签名。安全，但是效率低。
- 共享密钥加密（对称加密）：加密和解密用同一个密钥。不安全，密钥容易泄露，但效率高。

## 客户端与服务端交互
- 前提
客户端：持有CA的公钥，一般预制在浏览器或者操作系统中。
服务端：向CA申请数字证书，包含服务器公钥和域名、CA私钥签名等。
- 过程
  1. client向server发起请求，准备会话连接；
  2. server给client发送从CA获取的数字证书；
  3. client使用CA公钥（非对称加密）验证数字证书，并提取到server公钥（非对称加密）信息，同时生成会话密钥（对称加密）；
  4. client把会话密钥发给server，发送前会使用server公钥（非对称加密）对会话密钥加密；
  5. server使用自己的私钥（非对称加密）解密client的消息，获取到client生成的会话密钥；
  6. client和server都持有会话密钥，之后的消息通信使用会话密钥进行加密和解密；
- 加密方式的切换
非对称加密（更安全） → 对称加密（更高效）
在交互过程中，因为client和server都信任CA，通过CA密钥锁的保护把server公钥给到了client；接着使用server密钥锁的保护，client把会话密钥给到了server；最后client和server都持有共享密钥，借用共享密钥交互信息加密；利用非对称加密的安全性传递了对称加密的密钥给到对端，尽可能的提高信息传递的效率。