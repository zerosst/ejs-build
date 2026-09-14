# `ejs-neogeo-build` —— 用 EmulatorJS 的流水线编一个 NeoGeo 专用街机核心

**目的**：给出一个 ≤2MB 的 NeoGeo 核心，让小程序能跑 `kof97.zip` 这类 NeoGeo 游戏。

**为什么必须自己编**（详见 `docs/arcade-neogeo-feasibility.md`）：

| 问题 | 现状 |
|---|---|
| 核心带不带 kof97 驱动 | ✅ 带（fbneo / 上游 neogeo 都有） |
| 现成核心塞得进 2MB 分包吗 | ❌ fbneo 8.39MB、mame2003 5.03MB、mame2003_plus 5.39MB；**没有 NeoGeo 专用核心发布过**（CDN 各种命名全 404） |
| 能不能运行时下载核心 | ❌ `WXWebAssembly.instantiate(path)` 只吃**代码包内**路径（不接受 ArrayBuffer / wxfile / http） |
| 本机能不能编 | ❌ 本机没有 Docker、WSL 被安全策略拦、`perl`/`make`/`cmake` 全缺（`gamelist.pl` 必须要 perl） |

⇒ 只剩**借 Linux CI 编**。这里借 GitHub Actions 的免费 runner（公开仓库不计量分钟数）。

---

## 这个目录里的东西

| 文件 | 说明 | 来源 |
|---|---|---|
| `build.sh` | EJS 的构建驱动脚本（446 行）：clone `EmulatorJS/RetroArch` + `EmulatorJS/EmulatorJS` + 核心仓库 → `emmake make -f makefile.libretro platform=emscripten` 出 `.bc` → 丢进 `RetroArch/emulatorjs/core-temp/<变体>/` → `emmake ./build-emulatorjs.sh` 链出 wasm → 7z 打包成 `<name>-<变体>-wasm.data` | `EmulatorJS/build`（**未改动**） |
| `cores.json` | 核心清单，每个核心一条 JSON（`.data` 里的 `core.json` 就是它的副本）。**我们加了一条 `fbalpha2012_neogeo`** | 同上 + 我们插入 |
| `build.json` | `{minimumEJSVersion, version}`，会被打进 `.data` | 同上（当前 4.3.0 / 2.0.3） |
| `VERSION` | 构建工具版本号 | 同上 |
| `core-patch/makefile.libretro` | **上游 `libretro/fbalpha2012_neogeo` 的 makefile + EJS 的 21 行补丁**（由 `tools/apply_ejs_patch.py` 生成） | 上游 + 我们 |
| `core-patch/makefile.libretro.upstream` | 上游原文，便于对照 | 上游 |
| `.github/workflows/build-core.yml` | **最小可用版** CI（官方的跑在自托管 runner 上，fork 里跑不了） | 我们 |

## 那 21 行补丁是什么

EJS 的每个核心都是「上游 core 仓库 + 一小撮补丁」。把 `EmulatorJS/fbalpha2012_cps1` 和
`libretro/fbalpha2012_cps1` 的 makefile 一比，**差异只有 21 行**——这就是 EJS 对核心的全部改动：

1. `EMULATORJS_THREADS ?= 0`
2. emscripten 分支加 `EXTERNAL_ZLIB = 1`（不设会和 RetroArch 自带的 zlib **撞符号**）
3. emscripten 分支加 `EMULATORJS_THREADS=1 → -pthread`
4. `-O` 档：emscripten 走 `-O3 -DNDEBUG`

`tools/apply_ejs_patch.py` 把这 4 处照搬到 neogeo 的 makefile 上，并**打印 diff 供人工核对**
（落点找不到就直接报错，不静默产出半成品）。

## 怎么跑

**前置**：一个 GitHub 账号 + 一个 PAT（经典 token，勾 `repo` + `workflow`）。
把 token 存到文件（不要提交进任何仓库）：

```bash
mkdir -p ~/.workbuddy && printf '%s' 'ghp_你的token' > ~/.workbuddy/.gh_token && chmod 600 ~/.workbuddy/.gh_token
```

然后一条命令（幂等，可重复跑）：

```bash
python tools/ejs_neogeo_remote.py --token-file ~/.workbuddy/.gh_token
```

它会：① 建（或复用）`<你>/ejs-build` 并把本目录推上去 → ② 触发构建 → ③ 轮询到结束 →
④ 把 artifact 下到 `dist/arcade/_ejs/`。

只想跑某一步：`--step push|run|watch|fetch`。

**只建一个仓库，核心源码不进任何仓库** —— CI 在 runner 里
`git clone libretro/fbalpha2012_neogeo` 到 `compile/<name>/`，再把
`core-patch/makefile.libretro` 覆盖上去。这样做的原因：

- `build.sh` 里是 `if [ ! -d "$name" ]; then git clone …; fi` —— 目录已存在就**不会 clone**，直接用我们这份；
- 本机不必 clone 大仓库（GitHub 直连在国内可能不稳），重活都在 CI 里做；
- 少一个仓库要管；补丁内容随构建仓库一起被 review。

⚠️ 这也是**覆盖**而非 `patch` 打补丁：如果上游 makefile 以后大改，本目录那份会过时。
届时重跑 `python tools/apply_ejs_patch.py <新上游 makefile> core-patch/makefile.libretro` 更新即可。

## 拿到产物之后（阶段 1 的判据）

```bash
python tools/core_driver_probe.py dist/arcade/_ejs/core-fbalpha2012_neogeo/*.data kof97 --size
```

- **brotli ≤ 2MB** → 继续阶段 2（见 `docs/arcade-neogeo-feasibility.md`）
- 超了 → 先裁 `gamelist.txt`（把不玩的 NeoGeo 游戏标 `X`，`d_neogeo.cpp` 的 ROM 表会显著缩小）再编一轮

## 已知风险

1. **EJS 构建工具版本比我们手上的 CPS1 核心新**：现有 CPS1 核心的 `build.json` 是 `4.2.2 / 2.0.2`，
   现在官方是 `4.3.0 / 2.0.3`，`EmulatorJS/RetroArch` 默认分支也已更新 → **新胶水的形状可能不同**。
   影响：`tools/pack_arcade_core.py` 的补丁模式、`[arcade-mini-bridge]` 环境桥、小程序适配层都要重新验。
   验证手段现成：`test_arcade.js` 第 13 节（按微信包装方式装真胶水做正反例）+
   `.eval/arcade_mini_env.js --wxmodule*` 五层。
2. 构建时间：RetroArch 首次编译较久（估计 1~2 小时），CI 限 6 小时。
3. 真机内存：KOF97 解压后 62MB ROM + 41MB wasm —— 这是体积之外的第二大未知，得真机试。

## 许可

`build.sh` / `cores.json` / `build.json` / `VERSION` 来自 [EmulatorJS/build](https://github.com/EmulatorJS/build)（GPLv3）；
核心源码来自 [libretro/fbalpha2012_neogeo](https://github.com/libretro/fbalpha2012_neogeo)。
FBA 系列因版权原因曾下架，仅在本机自用；不要公开分发带 ROM 的产物。
