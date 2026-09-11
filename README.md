# test_config：Codex 补丁版自动构建与分发

这个分支只放工具，不含 Codex 源码。GitHub Actions 每天检查上游 `openai/codex` 的最新稳定 tag，
取出源码、应用 `patches/*.patch`、构建 macOS arm64 二进制，并发布到本仓库的 Releases。

- 版本号继承自上游 tag（上游在打 tag 的提交里已把版本写进 `codex-rs/Cargo.toml`），`codex --version` 与上游一致。
- 发布 tag 命名为 `test_config-v<版本号>`，对应的打过补丁的源码推送到分支 `release/test_config-v<版本号>`。
- 补丁应用失败或构建失败时自动开一个 issue。

## 补丁内容

`patches/0001-*.patch` 给 `codex-rs/cloud-config/src/bundle_loader.rs` 增加两个环境变量：

| 环境变量 | 作用 |
|---|---|
| `CODEX_CLOUD_CONFIG_BASE_URL_OVERRIDE` | 企业配置包改从这个地址拉取，例如 `https://codex-policy.example.com/backend-api`。其他 ChatGPT 后端请求不受影响。 |
| `CODEX_CLOUD_CONFIG_DISABLED` | 设为 `1` 或 `true` 时完全不拉配置包（灰度/测试开关），优先级高于上一项。 |

## 安装（员工）

```bash
curl -fsSL https://github.com/marvinli00/codex/releases/latest/download/codex-aarch64-apple-darwin.tar.gz | tar -xz
sudo mv codex-aarch64-apple-darwin /usr/local/bin/codex
xattr -d com.apple.quarantine /usr/local/bin/codex 2>/dev/null || true
codex --version
```

二进制未做 Apple 签名和公证，所以需要去掉 quarantine 属性。建议在系统级 `requirements.toml` 里设置
`check_for_update_on_startup = false`，避免官方更新提示。

## 手动触发

Actions 页面选择 "test_config sync and release" → Run workflow。可选参数：
`upstream_tag`（指定上游 tag）、`include_prerelease`（允许 alpha/beta）、`force`（已存在 release 也重建）。

## 更新补丁

在上游代码上改好后：

```bash
git format-patch -1 <commit> -o patches/
```

把生成的文件提交到本分支即可，工作流按文件名顺序 `git am --3way` 应用。

## 首次启用

1. fork 的 Actions 页面点击启用工作流（fork 默认关闭）。
2. 把仓库默认分支改为 `test_config`（定时任务只在默认分支上运行）：
   `gh repo edit marvinli00/codex --default-branch test_config`。
3. 手动运行一次工作流验证。
