# git 设置和取消代理

标签（空格分隔）： git

---

本地开启VPN后，GIt也需要设置代理，才能正常略过GFW，访问goole code等网站

## 设置代理

    git config --global http.proxy http://127.0.0.1:1080
    git config --global https.proxy https://127.0.0.1:1080
    git config --global http.proxy 'socks5://127.0.0.1:1080' 
    git config --global https.proxy 'socks5://127.0.0.1:1080'

## 取消代理

    git config --global --unset http.proxy
    git config --global --unset https.proxy