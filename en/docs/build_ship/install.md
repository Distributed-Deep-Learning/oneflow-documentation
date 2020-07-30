## 安装 OneFlow 发布版

通过以下命令，安装 OneFlow 稳定发布版：

```shell
python3 -m pip install oneflow
```


## 安装常见问题解决方案

* 如果提示失败，请尝试升级 `pip` ：
```shell
python3 -m pip install --upgrade pip
python3 -m pip install oneflow
```

* 如果提示没有写权限(`site-packages is not writeable`)，很可能是因为没有管理权限造成的，可以加上 `--user` 选项(安装 OneFlow 同理)：
```shell
python3 -m pip install --upgrade pip --user
python3 -m pip install oneflow --user
```

* 如果下载速度较慢，可以尝试使用国内镜像
```shell
python3 -m pip install oneflow -i https://pypi.douban.com/simple
```

## 从源码编译安装 OneFlow

如果你希望通过编译源码安装 OneFlow，可以参考 OneFlow源码仓库的[README](https://github.com/Oneflow-Inc/oneflow/blob/develop/README.md)，在编译 OneFlow 源码之前，强烈推荐先阅读 [Troubleshooting](https://github.com/Oneflow-Inc/oneflow/blob/develop/docs/source/troubleshooting.md)。

## 安装 OneFlow Nightly 版本

我们也提供了每天更新的 OneFlow Nightly 版本，适合想优先体验 OneFlow 新特性的朋友，不建议在生产环境下使用，通过以下命令安装：

```shell
python3 -m pip install --find-links https://oneflow-inc.github.io/nightly oneflow
```

## 交流QQ群
安装或使用过程遇到问题，欢迎入群与众多 OneFlow 爱好者共同讨论交流：

**加QQ群 331883 或扫描二维码**

![qq group](../contribute/imgs/qq_group.png)