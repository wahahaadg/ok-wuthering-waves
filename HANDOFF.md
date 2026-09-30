# 鸣潮声骸堆叠适配：跨电脑交接

日期：2026-09-30（北京时间）

## 项目与当前修改

- 上游：https://github.com/ok-oldking/ok-wuthering-waves
- 基础版本：`master`，提交 `61bfa64`。
- 本次修改：`src/task/EnhanceEchoTask.py` 的 `run()` 外层循环。
- 当前源码完整，但本地采用浅克隆，未下载全部历史。
- 未创建发布标签，未发布安装包。

游戏新增声骸堆叠后，强化过的声骸退出培养界面时仍保持选中。旧代码下一轮直接检查当前声骸是否零级，可能误报“无可强化声骸”并结束任务，即使背包第一位的堆叠中仍有未强化声骸。

修复后每轮执行顺序：

1. 识别“培养”按钮，确认在背包界面。
2. 点击背包第一格中心，相对坐标 `(0.15, 0.20)`。
3. 等待 0.5 秒，让右侧详情刷新。
4. 重新识别“培养”按钮；找不到时明确报错。
5. 执行原有零级判断，再进入培养、强化与调谐。

弃置与成功上锁都回到同一个外层循环，因此两条路径均会重新选中第一格。词条筛选、强化、弃置、上锁和成功暂停逻辑保持不变。

## 相关代码与界面假设

- `find_echo_enhance()`：识别右下角“培养”。
- `is_0_level()`：在右侧详情的相对区域 `(0.65, 0.35, 1, 0.57)` 识别“声骸技能”，通过布局间接判断零级；并不读取卡片右下角数字。
- `find_add_mat()`：识别“阶段放入”。
- `check_echo_stats()`：判断双爆、首条双爆数值、有效词条及剩余孔位。
- `trash_and_esc()`：标记弃置并返回背包。
- `lock_and_esc()`：上锁，可暂停，然后返回背包。
- `esc()`：反复按 Esc，直到识别到背包“培养”按钮。
- `config.py`：在 `onetime_tasks` 中注册 `EnhanceEchoTask`。
- `tests/TestEnchaneEcho.py`：零级、材料按钮、确认按钮的截图测试。
- `tests/TestEnhanceEchoStatusBox.py`：弃置和上锁状态识别区域测试。

新版堆叠卡片左下角是堆叠标志，右下角数字（例如 60）是数量，不是等级。本次修改不读取该数字。第一格点击坐标来自仓库旧测试截图 `tests/images/echo_enhance.png`；用户提供的新截图仅包含单张卡片，尚不能确认新版完整背包坐标。右侧详情的零级识别区域仍需实测。

任务仍要求：先在背包过滤待强化声骸，并按等级从零级开始升序排列。第一格应处于列表顶部；当前修改不会主动滚动列表。

## 验证情况与下一步

已完成：

- Python AST 语法检查。
- 对实际 `run()` 方法使用模拟界面进行验证：弃置和上锁后均先重新选中第一格，再检查是否还有零级声骸；进入培养使用重新识别的按钮。
- `git diff --check`。

尚未完成：

- 本机没有 `.venv`，全局 Python 为 3.13.2，缺少 `ok-script`，因此未运行依赖框架的原生测试。
- 未在游戏中运行，未验证点击坐标、0.5 秒刷新时长、堆叠耗尽和新版右侧详情布局。
- 未生成 EXE 或安装包。

另一台电脑建议先测试：弃置后继续强化堆叠、成功上锁后继续、成功暂停并恢复、堆叠耗尽后切换到下一格或正常结束。若不能选中第一格，先用完整新版背包截图校准坐标；若仍误判零级，检查右侧详情 OCR 区域。

## 从源码运行（Windows PowerShell）

开发指南 `docs/development/contributing.md` 明确使用 Python 3.12，官方发布流程也使用 3.12。虽然 README 的最低版本说明更宽泛，跨电脑配置应以 Python 3.12 为准。

先安装 64 位 Python 3.12 和 Git，在需要的目录克隆自己 GitHub 上的仓库并进入仓库根目录。以下命令假设 Python Launcher 能找到 3.12：

```powershell
py -3.12 -m venv .venv
```

本机 Clash 代理地址为 `http://127.0.0.1:7897`。另一台电脑应先检查自己的 Clash 端口，按需调整下面的地址。Git 的代理配置只作用于当前仓库：

```powershell
git config http.proxy http://127.0.0.1:7897
.\.venv\Scripts\python.exe -m pip install --proxy http://127.0.0.1:7897 -r requirements.txt
```

启动普通模式或调试模式（二选一）：

```powershell
.\.venv\Scripts\python.exe main.py
.\.venv\Scripts\python.exe main_debug.py
```

`main.py` 创建 `OK(config)` 并启动；`main_debug.py` 另外启用调试配置以及 `PYAPPIFY_PYTHON_TEST=1`。从源码运行会直接加载本地修改，修改后重启程序即可。

开发测试依赖及相关测试：

```powershell
.\.venv\Scripts\python.exe -m pip install --proxy http://127.0.0.1:7897 ".[dev]"
.\.venv\Scripts\python.exe -m unittest tests.TestEnchaneEcho tests.TestEnhanceEchoStatusBox
```

遵循 `AGENTS.md`：存在仓库本地虚拟环境时，所有 Python 命令优先直接调用 `.venv/Scripts/python.exe`，不要依赖激活环境。

## EXE 与完整安装包

项目使用 PyAppify：

- `pyappify.yml`：应用名 `ok-ww`，入口 `main.py`，依赖 `requirements.txt`，China 配置指定 Python 3.12。
- `.github/workflows/build.yml`：实际发布流程，Windows + Python 3.12 + `ok-oldking/pyappify-action@master`，产物目录 `pyappify_dist`。
- `setup.py`：Python 包与可选 Cython 扩展构建入口，不是 EXE 打包命令。
- `.github/workflows/sign_exe.yml`：引用当前检出中不存在的 `src-win-launcher`，不应作为本次个人打包入口。

PyAppify 的 EXE 是启动器：准备独立 Python、安装依赖、获取源码，然后启动应用。可下载预编译启动器并搭配 `pyappify.yml`；完整离线分发需要包含源码、Python 和依赖。

重要：当前 `pyappify.yml` 的 China/Global `git_url` 指向作者的更新仓库。直接沿用会下载官方版本，无法保证包含本次修改。个人分发需要将其指向自己的代码或更新仓库，并为启动器的版本管理准备版本标签。当前交接只上传源码，不修改更新地址，也不创建发布标签。

官方完整发布流程还包含同步到作者的更新仓库、SignPath 签名和发布 Release，依赖作者的 secrets。个人 Fork 不应直接运行该完整流程；先创建独立的手动构建工作流，使用 PyAppify Action 并上传构建产物，验证后再考虑发布。

官方资料（2026-09-30 已通过 Clash 核对）：

- https://github.com/ok-oldking/pyappify
- https://github.com/ok-oldking/pyappify-action

本次没有验证通用的本地 PyAppify CLI 打包命令，不应假设 `python -m pyappify` 或 `setup.py` 会直接生成可用 EXE。

## 在另一台电脑接续

1. 从自己的 GitHub 仓库克隆项目，查看本文件。
2. 安装 Python 3.12，建立 `.venv` 并安装依赖。
3. 使用 `main_debug.py` 启动，在新版背包过滤并按等级升序排序，确认列表在顶部。
4. 先验证声骸堆叠适配，再决定是否完善测试或构建安装包。
5. 日志、截图、虚拟环境和个人配置留在本地，不要提交到 GitHub。

与上游同步时，应保持自己的 GitHub 仓库为 `origin`，作者仓库为 `upstream`；GitHub 上传后的实际 remote 和提交信息以本地 `git remote -v`、`git log -1` 为准。
