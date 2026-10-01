# `_sync/` —— 插件同步清单说明

这个目录只放**同步用的清单**，不影响编译（OpenWrt 的 feed 只认子目录里的 `Makefile`，
`_sync/` 里没有 `Makefile`，所以编译时会被忽略）。

## 它是干什么的

仓库根目录的 `.github/workflows/sync.yml` 会**每 2 天**（北京时间 18:30）自动运行一次，
按本分支的 `plugins.txt`，把每个插件的 **原始上游仓库**的最新代码拉下来，覆盖到对应的插件目录。

- 想立刻同步：仓库 → **Actions** → 左侧 **同步插件** → **Run workflow**
  - `BRANCHES` 留空 = 全部 11 个分支
  - `DRY_RUN` 填 `1` = **只对比不推送**，先看报告（推荐先这样跑一次）
  - `ONLY_DIR` 填某个目录名 = 只同步那一个插件（调试用）

## `plugins.txt` 格式

一行一个插件，用 `|` 分隔，**6 个字段**：

```
本地目录名|上游仓库|上游分支|上游里的子目录|本地里的目标子目录|备注
```

| 字段 | 说明 |
| --- | --- |
| 1 本地目录名 | 本分支根目录下的插件目录，例如 `luci-app-passwall` |
| 2 上游仓库 | `账号/仓库`，例如 `xiaorouji/openwrt-passwall`；或 `@kenzok8` / `@frozen` |
| 3 上游分支 | 例如 `main`、`master`，留空 = 自动探测上游默认分支 |
| 4 上游里的子目录 | 插件在上游仓库里的位置；**留空 = 上游仓库根目录就是插件** |
| 5 本地里的目标子目录 | 一般留空；只有像 `relevance` 这种"一个目录装好几套东西"才用 |
| 6 备注 | 随便写，只给人看 |

`#` 开头是注释，空行忽略。

**取值规则**（脚本就是这么执行的）：

- 源 = `上游仓库/第4字段`，第 4 字段留空 = **上游仓库根目录**
- 目标 = `本分支/第1字段/第5字段`，第 5 字段留空 = 第 1 字段那个目录本身
- 第 2 字段写 `@kenzok8` 时，如果第 4 字段留空，就取 kenzok8 里**和第 1 字段同名**（或第 5 字段同名）的目录
- 第 3 字段留空时会自动探测上游的默认分支

### 三种写法

**① 正常：从原始上游同步**（推荐，永远拿最新）

```
luci-app-pushbot|zzsj0928/luci-app-pushbot|master|||作者的原始仓库
luci-app-passwall|xiaorouji/openwrt-passwall|main|luci-app-passwall||插件在上游的子目录里
luci-app-amlogic|ophub/luci-app-amlogic|main|||上游仓库根目录就是插件
```

**② 找不到原始上游：从 kenzok8 兜底**

第 2 个字段写 `@kenzok8`，第 4 字段留空就自动取同名目录：

```
luci-app-fastnet|@kenzok8||||上游没找到，先用 kenzok8 兜底
relevance|@kenzok8||adguardhome|adguardhome|把 kenzok8 的 adguardhome 放进 relevance/adguardhome
```

**③ 完全不更新：冻结现状**

第 2 个字段写 `@frozen`，同步时直接跳过（保留现在的内容）：

```
luci-app-xxxx|@frozen||||这插件已经没人维护了，保持现状
```

## 从哪拿代码：`_sync/settings.ini` 开关（默认 kenzok8）

**默认分支**（现在是 `Lede`）上的 `_sync/settings.ini` 有一个总开关，控制所有分支：

```ini
SYNC_SOURCE="kenzok8"     # 默认：全部从 kenzok8/small-package 取同名目录（最稳）
# SYNC_SOURCE="upstream"  # 改成这个：优先按各分支 plugins.txt 里的原始上游取
```

* `kenzok8`：内容和你现在仓库里的一致（实测同一文件的 git blob SHA 逐一相同），它每天更新 5~8 次
  * **kenzok8 里没有的目录会自动回退到它的原始上游**（例如 `luci-app-advancedplus`、`luci-app-partexp`
    这类 kenzok8 没收录的），所以默认模式下不会有插件被漏掉
* `upstream`：作者还在维护的插件能早几小时拿到；清单里写 `@kenzok8` 或验证不过的仍走 kenzok8
* **开关只影响"从哪拿"，不会改动 `plugins.txt` 里记录的原始上游清单**，随时可以来回切

改完保存 → Actions → 同步插件 → Run workflow 即生效（想先看效果就填 `DRY_RUN=1`）。

## 我要新增一个插件，怎么做

1. 切到你要加插件的分支（例如 `Immortalwrt`）
2. 打开 `_sync/plugins.txt` → 点右上角铅笔编辑
3. 在**合适的位置加一行**（新加的插件必须先在分支根目录里存在，否则同步时会新建空目录——
   也可以先加目录再跑同步，脚本会自动 `mkdir`）
4. 提交到**当前分支**
5. Actions → 同步插件 → Run workflow（`BRANCHES` 填这个分支名，`DRY_RUN=1` 先看效果）→ 没问题再 `DRY_RUN=0` 跑一次

> 只要知道插件的 GitHub 地址，直接照抄格式填第 1、2、3 字段就行；不知道原始地址就写 `@kenzok8`。

## 同步脚本的边界（放心的地方）

- **只动清单里列出的目录**，根目录的 `LICENSE`、`README.md`、`_sync/` 一律不碰
- 每个目录内部用 `rsync -a --delete`，所以上游删掉的文件本地也会删掉
- 同步完会自动做一次**引用改写**：把 `281677160/*`、`danshui-git/*` 改回 `authon/Mine-*`
  （万一上游内容里带着老账号地址，不会被带回本仓库）
- 单个插件失败（上游拉不到 / 上游没这个目录）只记进报告，不影响其它插件
- 跑完在 Actions 的 **Summary** 页有一张「分支 / 插件 / 上游 / 结果」的表
