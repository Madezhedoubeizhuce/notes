# 设置淘宝镜像

## 临时使用

``` shell
npm --registry https://registry.npm.taobao.org install express
```

## 使用cnpm

```shell
npm install -g cnpm - -registry=https://registry.npm.taobao.org
cnpm -express
```

## 长期使用

```shell
npm config set registry https://registry.npm.taobao.org
```

```shell
npm config get registry
```

