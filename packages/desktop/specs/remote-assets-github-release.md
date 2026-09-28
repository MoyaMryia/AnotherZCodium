# 远程资源：GitHub Release 发布与自建源加载

审计版不把 remote runtime 打进安装包，也不默认连官方 CDN。发布方可以把 mock-cdn 的产物以扁平布局发布到 GitHub Release，客户端用显式自建源（`ZCODE_REMOTE_ASSET_CDN_BASE_URL`）加载。源解析与官方开关的解耦见 [official-service-switches.md](../../services/specs/official-service-switches.md)。

## 所有权

- **构建**：`scripts/prepare-prebuilds.mjs`（`pnpm prepare:remote-assets`）产出 `packages/desktop/mock-cdn` 的目录式布局：`releases/<version>/manifest-<arch>.json` + `components/<platform>/<component>/<version+sha>.tar.gz`。
- **发布布局转换**：`scripts/publish-remote-assets.mjs` 是唯一 owner，把目录式布局转换成 GitHub Release 扁平资产，不改动 mock-cdn。产物是 `manifest-<arch>.json` + `<platformArch>__<componentId>__<safeVersion>.tar.gz`。
- **消费方**：`@zcode/server` 的 remote asset 加载（`remoteAssetCache.ts`）按 manifest 的 `artifactPath` 拼 `<base>/<artifactPath>` 下载，不感知托管介质；sha256 校验针对文件内容，与文件名无关。

## 扁平命名契约

- GitHub Release 的 asset 名不允许 `/`，组件必须压成单段文件名。
- 版本里的 `+` 统一替换为 `-`：客户端会把 `+` 编码成 `%2B`，不同托管端的解码行为不一致；替换后下载 URL 与 asset 名完全一致。
- 命名：`<platformArch>__<componentId>__<version(+ → -)>.tar.gz`，例如 `linux-x64__server-bundle__v3.14.4-ef831e13132e.tar.gz`。
- manifest 其余字段（`id` / `version` / `sha256` / `mount`）保持原值，只重写 `artifactPath`。
- 复制前对源文件复验 `sha256`：发布的是内容寻址制品，不能把损坏或被替换的文件以“同名可信制品”发出去。
- 脚本在开始前重建输出目录，并拒绝输出目录包含源目录或仓库根。

## tag 与寻址假设

- 发布到 app release 的 tag（`v<version>`，如 `v3.14.4`）。
- 客户端 base = `https://github.com/<owner>/<repo>/releases/download/<tag>`：
  - manifest 候选：`<base>/<version>/manifest-<arch>.json`（404，被加载器跳过）→ `<base>/manifest-<arch>.json`（命中）。
  - 组件候选：`<base>/<artifactPath>`（扁平名，直接命中；tag 带 `v` 前缀时不会产生父级探测）。
  - manifest 的 `appVersion` 与当前 app 版本不一致时，加载器按原有校验拒绝，避免把旧版本资源装入新客户端。

## CI

- release workflow 的 `remote-assets` job：`pnpm prepare:remote-assets` → `node scripts/publish-remote-assets.mjs --out dist/remote-assets-github` → `gh release upload <tag> --clobber`（带重试）。
- 失败语义与 CLI / 桌面包一致：任一产物上传失败则 release 保持 draft；`build_artifacts=false` 时跳过。

## 验收

1. 发布脚本单测：扁平命名（无 `/`、无 `+`）、manifest `artifactPath` 重写、sha256 不变、多平台发现与 `--platforms` 过滤、缺失组件与 sha256 不匹配报错。
2. 加载链路测试：本地 HTTP 服务扁平布局，`ensureRemoteReleaseDirFromCdn` 能按 manifest 下载、校验并物化到 `releases/<version>/<platform>/<mount>`。
3. `pnpm typecheck`、`pnpm lint`、架构检查通过；release workflow YAML 语法合法。
