```
host-192-168-2-2:/home/mtuser # ssh mtuser@192.168.2.1
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ED25519 key sent by the remote host is
SHA256:9b4pMgYiiReTDoYz7DkZoDJt3CmNowpriE5PzhUc8jE.
Please contact your system administrator.
Add correct host key in /root/.ssh/known_hosts to get rid of this message.
Offending ED25519 key in /root/.ssh/known_hosts:1
Host key for 192.168.2.1 has changed and you have requested strict checking.
Host key verification failed.

```

---

解决方案

```
ssh-keygen -R 192.168.2.1
```

#ssh主机身份校验冲突 #ssh连接失败