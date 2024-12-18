第一次执行`mise install`时没有任何效果，查了一圈发现需要先执行`mise trust`后mise.toml才会生效。

所以执行顺序是：

```bash
git clone https://github.com/ethereum-optimism/optimism
cd optimism
mise trust
mise install --verbose
```

但mise处理"cargo:just" = "1.37.0"时有个报错，根因是没有自动安装cc，详见[此处](https://github.com/jdx/mise/issues/3662)。

手动执行`apt install gcc`后该问题解决。

如果想让`mise.toml`中的tool全局有效：

```bash
cp mise.toml ~/.config/mise/conf.d/extras.toml
```
([source](https://mise.jdx.dev/configuration.html))