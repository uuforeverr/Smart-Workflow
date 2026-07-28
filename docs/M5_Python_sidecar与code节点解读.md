# M5 Python sidecar 与 code 节点解读

本文用于精读 Smart-Workflow 的 M5 代码，覆盖：

- `sidecar/main.py`：FastAPI sidecar，暴露 `/run/python-code`
- `sidecar/runner.py`：子进程入口，真正 exec 用户代码
- `sidecar/contract/code_run.golden.json`：Go↔Python 共享契约 fixture
- `internal/engine/nodes/code.go`：Go 侧 code 节点执行器（M4 已见，本文补执行视角）
- `sidecar/test_runner_sanitize.py` / `contract/test_contract.py`：Python 侧测试
- `internal/engine/nodes/code_test.go` / `contract_test.go`：Go 侧测试
- `deployments/Dockerfile.sidecar`：sidecar 容器化

M5 的核心目标：

```text
让工作流能跑“用户写的 Python 代码”，但不能让这段代码搞垮引擎：
Go 引擎不自己执行 Python，而是通过 HTTP 把 code + inputs 交给一个独立的
Python sidecar；sidecar 再 fork 一个子进程来 exec 用户代码，
用“进程隔离 + 超时 kill + 输出限流 + 错误脱敏”把风险关在子进程里。
```

先给结论：

```text
code 节点是第一个“执行不可信代码”的节点，也是第一个跨语言、跨进程的节点。
M4 的调度器只管把 code 节点派发出去，具体怎么安全地跑，全在 M5。
关键词：三层进程边界（Go 引擎 / Python sidecar / 用户子进程）。
```

---

## 1. 先回答几个最容易混淆的问题

### 1.1 用户的 Python 代码到底在哪一层执行？为什么要三层？

在最外面的**子进程**里。一次 code 节点执行跨越三个进程：

```text
[进程①] Go 引擎（scheduler worker）
   |  HTTP POST /run/python-code  {code, inputs, timeout_sec}
   v
[进程②] Python sidecar（FastAPI / uvicorn，常驻）
   |  subprocess.run([python, runner.py], input=payload)
   v
[进程③] Python runner 子进程（一次性，用完即弃）
        exec(用户代码)  <- 用户代码只在这里跑
```

为什么要三层，而不是让 Go 直接 `exec` 或让 sidecar 直接 `exec`？

```text
① Go 不执行 Python：语言隔离。Go 引擎不引入 Python 运行时，崩溃/内存泄漏不波及引擎。
② sidecar 不亲自 exec：故障隔离。用户代码若死循环/OOM/段错误，
   死的是一次性子进程③，常驻的 sidecar②不受影响，能继续服务下一个请求。
③ 子进程一次性：状态隔离。每次执行全新进程，用户代码改不了全局状态、
   污染不了下一次执行。
```

一句话：**常驻的要稳，危险的要一次性**。sidecar 常驻但不碰用户代码，子进程碰用户代码但用完即弃。

### 1.2 这是“沙箱”吗？安全边界到底在哪？

**不是语言级/系统级真沙箱**。这点 plan.md 和代码注释都反复强调（技术债 TD-5）：

```text
—— 注意：这是"进程隔离+超时"，非语言级/系统级真沙箱（见技术债 TD-5）
```

`runner.py` 里那行 `exec` 的注释也直说：

```python
exec(compiled, sandbox)  # noqa: S102 - 进程隔离 + 超时 kill 提供隔离，非语言级沙箱
```

当前提供的边界是四条“工程护栏”，不是内核级隔离：

```text
1. 进程隔离：用户代码在独立子进程，崩溃不影响 sidecar
2. 超时 kill：subprocess.run(timeout=...) 到点强杀子进程
3. 输出限流：日志 64KB、输出 256KB 上限，防刷爆内存/管道
4. 错误脱敏：对外不泄露服务器绝对路径（TD-6）
```

它**挡不住**的（真沙箱才能做的）：

```text
用户代码 import os 读文件、发网络请求、消耗 CPU 到超时上限……
```

所以名字叫 `sandbox`（`runner.py` 里的变量名），但心里要清楚它只是“受控命名空间 + 进程护栏”。真沙箱（seccomp/gVisor/容器逐次隔离）是明确的后续增强（第 9 节）。

### 1.3 用户代码怎么拿到输入、怎么返回输出？

输入通过一个注入的全局变量 `inputs`，输出通过一个注入的函数 `sink()`。这套心智对齐 PaiFlow 的 code 节点。

看 `runner.py` 构造的命名空间：

```python
sandbox = {
    "__name__": "__user__",
    "inputs": inputs,   # 注入输入：用户直接读 inputs['a']
    "sink": sink,       # 注入出口：用户调 sink({...}) 提交输出
}
exec(compiled, sandbox)
```

用户代码就长这样（来自集成测试）：

```python
total = sum(inputs['nums'])
print('computed', total)          # print 会被隔离进日志，不污染结果
sink({'total': total, 'n': len(inputs['nums'])})
```

`sink` 的定义（可多次调用，累积合并）：

```python
def sink(obj):
    if not isinstance(obj, dict):
        raise TypeError("sink() expects a dict")
    collected.update(obj)         # 多次 sink 合并进同一个 dict
```

所以：

```text
读输入：inputs['key']            <- Go 侧 ec.Inputs 一路传下来的
写输出：sink({'key': value})     <- 收集进 collected，最后序列化回传
调试打印：print(...)             <- 进日志缓冲，不进结果通道
```

### 1.4 为什么用户的 `print` 不会弄坏返回结果？

因为 `runner.py` 玩了一手“抢占 fd 1”：先把真实 stdout（文件描述符 1）复制出来当**唯一结果通道**，再把 Python 的 `sys.stdout`/`sys.stderr` 换成内存缓冲区。

```python
result_fd = os.dup(1)      # ① 先复制真实 stdout，留作结果专用通道
raw = sys.stdin.read()
log_buf = io.StringIO()
sys.stdout = log_buf       # ② 用户 print 从此写进内存缓冲
sys.stderr = log_buf
```

于是：

```text
用户 print("debug")     -> 写进 log_buf（内存），最终打包进 result 的 logs 字段
runner 写最终 JSON      -> os.write(result_fd, ...) 直接写原始 fd 1
```

两条通道彻底分开。用户在代码里 `print` 再多、甚至 `print` 一段假 JSON，都进不了结果通道，sidecar 解析 stdout 时只会读到 runner 自己写的那一行 JSON。这是保证“结果通道纯净”的关键技巧。

结果的写回用 `emit`：

```python
def emit(payload: dict) -> None:
    payload["logs"] = _truncate(log_buf.getvalue(), MAX_LOG_BYTES)  # 顺带塞日志
    blob = json.dumps(payload, ensure_ascii=False, default=str)
    os.write(result_fd, blob.encode("utf-8", "replace"))           # 走专用 fd
```

### 1.5 “错误脱敏”是什么意思？为什么重要？

用户代码抛异常时，完整的 Python traceback 会包含**服务器上的绝对路径**（比如 `runner.py` 的部署路径）。直接把它回给调用方，等于泄露服务器目录结构。TD-6 脱敏就是：**对外只保留异常类型、消息和用户代码帧，裁掉框架帧**。

看 `_sanitize_error`（`runner.py`）：

```python
def _sanitize_error(exc):
    head = f"{type(exc).__name__}: {exc}"   # 保留：ValueError: boom
    user_frames = []
    tb = exc.__traceback__
    while tb is not None:
        filename = tb.tb_frame.f_code.co_filename
        if filename == "<user_code>":        # 只保留用户代码帧
            user_frames.append(f'  File "<user_code>", line {tb.tb_lineno}')
        tb = tb.tb_next
    ...
```

这里有个精妙配合：编译用户代码时故意把文件名写成 `"<user_code>"`：

```python
compiled = compile(code, "<user_code>", "exec")
```

于是 traceback 里用户代码的帧文件名就是 `<user_code>`，脱敏时靠这个标记精准区分“用户帧”和“框架帧”。

对外与对内两条信息流：

```text
对外 message：ValueError: boom
              File "<user_code>", line 2      <- 只有类型+消息+用户行号
对内 logs：   完整 traceback（含 runner.py 绝对路径，服务端排障用）
```

完整 traceback 没有丢，而是写进 `logs`（服务端可见），对外的 `message` 才脱敏。兼顾了“对外安全”和“对内可排障”。

### 1.6 code 节点失败时，Go 引擎怎么知道是哪种失败？

靠**分层的错误码**。从子进程到 Go，每一层都把失败翻译成明确的 code，不会糊成一句“执行失败”。

```text
runner.py（子进程内）产出：
  SYNTAX_ERROR            用户代码语法错
  RUNTIME_ERROR           用户代码运行时抛异常（已脱敏）
  OUTPUT_NOT_SERIALIZABLE sink 的输出不能 JSON 化
  OUTPUT_TOO_LARGE        输出超 256KB
  BAD_REQUEST             请求 JSON 坏 / inputs 不是对象

main.py（sidecar）产出：
  TIMEOUT                 子进程超时被 kill
  RUNNER_CRASHED          子进程没写回结果（多半 OOM/被信号杀）
  BAD_RUNNER_OUTPUT       子进程输出无法解析
  BAD_REQUEST             code 为空

code.go（Go 引擎）：
  把上面任一 ok=false 翻成 Go error，带上 code+message，标记节点 failed
```

M4 讲过节点失败即中断整图。M5 的价值在于：中断时错误信息是**结构化、可归因**的——Agent 能据此判断“是我代码写错了（RUNTIME_ERROR）还是超时了（TIMEOUT）”。这契合你 memory 里“错误分类导出、精细化归因”的偏好。

---

## 2. M5 的整体架构

一次 code 节点执行的完整链路：

```text
M4 scheduler worker
   |  execNode -> registry.Get("code") -> CodeExecutor.Execute
   v
CodeExecutor（code.go）                       [Go 进程]
   |  POST {code, inputs, timeout_sec}
   |  http://sidecar:8090/run/python-code
   v
run_python_code（main.py）                    [Python sidecar 常驻进程]
   |  校验 code 非空 + 夹取 timeout（<=60s）
   |  subprocess.run([python, runner.py], input=payload, timeout=...)
   v
runner.main（runner.py）                       [一次性子进程]
   |  抢占 fd1 -> 隔离 print -> exec(用户代码) -> sink 收集 -> 严格序列化
   |  os.write(result_fd, 单行 JSON)
   v
main.py 解析子进程 stdout -> 包成 envelope -> HTTP 响应
   v
CodeExecutor 解析 envelope -> NodeResult / error -> 回 M4 调度器
```

核心文件与职责：

```text
code.go       Go 侧执行器：拼请求、发 HTTP、解 envelope、翻 error（M4 已建）
main.py       sidecar 门面：HTTP 契约、超时护栏、子进程编排、错误码归一
runner.py     子进程内核：命名空间隔离、fd 抢占、exec、脱敏、限流、严格序列化
golden.json   Go↔Python 共享契约，双向锁字段名
```

一句话概括分层：

```text
code.go   管“怎么把活儿发出去、怎么收结果”。
main.py   管“怎么安全地起子进程、超时了怎么办、崩了怎么办”。
runner.py 管“怎么在子进程里干净地跑用户代码”。
```

---

## 3. 为什么 code 节点要用 sidecar，而不是 Go 内嵌？

这是 M5 最根本的设计决策。三个理由：

```text
1. 语言匹配：用户写的是 Python。Go 没法直接跑 Python，硬嵌一个 Python 解释器
   （如 cgo + libpython）会把 Python 的 GIL、内存模型、崩溃风险全带进 Go 引擎进程。
2. 故障隔离：用户代码是不可信的。它可能死循环、OOM、段错误。这些风险必须关在
   一个“可以随时杀掉、杀掉也不影响别人”的进程里——独立 sidecar + 一次性子进程。
3. 契约清晰：Go 和 Python 之间只有一个 HTTP 接口 + 一份 JSON 契约。
   两侧独立部署、独立扩缩容、独立升级，耦合面只有那一份 envelope。
```

代价是多了一次网络往返和一次子进程 fork。但对“执行不可信代码”这种场景，隔离性远比这点开销重要。

这也解释了 M4 里 `ApplicationExecutor` 为什么 `kind=python` 直接复用 `CodeExecutor`——因为“跑 Python”这件事已经被 M5 收敛成一个干净的能力，application 节点不必重造。

---

## 4. sidecar 门面：main.py

源码：`sidecar/main.py`

### 4.1 统一响应契约

sidecar 的响应对齐 Go 侧 `httpx.Envelope`：

```python
def ok(data):
    return {"ok": True, "data": data}

def fail(code, message, details=None):
    return {"ok": False, "error": {"code": code, "message": message, "details": details}}
```

```text
成功: {"ok": true,  "data": {...}}
失败: {"ok": false, "error": {"code": "...", "message": "...", "details": {...}}}
```

整个平台（Go API、CLI、sidecar）用同一套 envelope，这是你项目里“统一 JSON envelope + 结构化错误”规范的一部分。

### 4.2 请求模型与超时护栏

```python
class CodeRunRequest(BaseModel):
    code: str
    inputs: dict = {}
    timeout_sec: int = DEFAULT_TIMEOUT_SEC   # 30

MAX_TIMEOUT_SEC = 60
DEFAULT_TIMEOUT_SEC = 30
```

超时是“双向夹取”的：

```python
timeout = req.timeout_sec if req.timeout_sec > 0 else DEFAULT_TIMEOUT_SEC
timeout = min(timeout, MAX_TIMEOUT_SEC)      # 请求可申请更短，但不得超过硬上限
```

```text
请求没给或给 <=0  -> 用默认 30s
请求给了正数      -> 用它，但不得超过硬上限 60s
```

这是防御性设计：不信任调用方传的 timeout，避免有人申请 `timeout=999999` 把子进程挂到天荒地老。

### 4.3 子进程编排与三种失败

核心就是一句 `subprocess.run`，但错误处理很讲究：

```python
payload = json.dumps({"code": req.code, "inputs": req.inputs}, ensure_ascii=False)
try:
    proc = subprocess.run(
        [sys.executable, RUNNER_PATH],   # 用同一个 python 起 runner.py
        input=payload.encode("utf-8"),   # 通过 stdin 喂 payload
        capture_output=True,
        timeout=timeout,                 # 到点强杀
    )
except subprocess.TimeoutExpired:
    return fail("TIMEOUT", f"python code exceeded {timeout}s", {"timeout_sec": timeout})
```

超时之后，`main.py` 还要处理子进程“活着回来但结果不对”的情况：

```text
1. proc.stdout 为空       -> RUNNER_CRASHED（子进程没写结果，多半 OOM/被信号杀）
2. stdout 不是合法 JSON   -> BAD_RUNNER_OUTPUT
3. 解析出 ok=true         -> 成功，搬 outputs/logs 进 data
4. 解析出 ok=false        -> 把 runner 的 error.code/message 透传出去
```

`RUNNER_CRASHED` 分支还会带上子进程的 returncode 和截断的 stderr，方便排障：

```python
if not proc.stdout:
    stderr = proc.stderr.decode("utf-8", "replace")[:2048]
    return fail("RUNNER_CRASHED", "runner produced no result", {
        "returncode": proc.returncode,
        "stderr": stderr,
    })
```

注意分工：**超时 kill 是 sidecar（main.py）的责任**，因为只有父进程能对子进程设 timeout；**exec 用户代码是 runner 的责任**。两者不越界。

---

## 5. 子进程内核：runner.py

源码：`sidecar/runner.py`

这是 M5 最精华的文件。它在一个一次性子进程里，把“安全地跑一段用户代码”拆成六个动作。

### 5.1 动作一：抢占结果通道，隔离用户输出

已在 1.4 讲过，回顾核心：

```python
result_fd = os.dup(1)      # 复制真实 stdout 当结果专用通道
sys.stdout = log_buf       # 用户 print 改道进内存缓冲
sys.stderr = log_buf
```

### 5.2 动作二：读入并校验请求

```python
raw = sys.stdin.read()
try:
    req = json.loads(raw) if raw.strip() else {}
except json.JSONDecodeError as exc:
    emit({"ok": False, "error": {"code": "BAD_REQUEST", ...}})
    return

inputs = req.get("inputs", {})
if not isinstance(inputs, dict):
    emit({"ok": False, "error": {"code": "BAD_REQUEST", "message": "inputs must be an object"}})
    return
```

请求从 **stdin** 读（不是命令行参数）——因为代码可能很长，走 stdin 不受命令行长度限制，也不会被 shell 转义搞坏。

### 5.3 动作三：构造受控命名空间

```python
sandbox = {
    "__name__": "__user__",
    "inputs": inputs,
    "sink": sink,
}
```

只注入三样东西。用户代码看不到 runner 的其它变量（如 `result_fd`、`log_buf`）——虽然 Python 的 exec 命名空间不是强隔离，但至少不主动把内部句柄递给用户。

### 5.4 动作四：编译 + 执行，分类捕获异常

```python
try:
    compiled = compile(code, "<user_code>", "exec")   # 文件名标记，供脱敏识别
    exec(compiled, sandbox)
except SyntaxError as exc:
    emit({"ok": False, "error": {"code": "SYNTAX_ERROR", ...}})
    return
except BaseException as exc:   # 用户代码任何异常都要标准化回传
    full_tb = traceback.format_exc()
    print(full_tb, file=sys.stderr)   # 完整栈进 logs（已重定向）
    safe_msg = _sanitize_error(exc)   # 对外脱敏
    emit({"ok": False, "error": {"code": "RUNTIME_ERROR", "message": safe_msg}})
    return
```

两点值得注意：

```text
1. 语法错和运行时错分开：SYNTAX_ERROR vs RUNTIME_ERROR，Agent 能区分“写错了”和“跑挂了”。
2. 捕获 BaseException 而非 Exception：连 SystemExit/KeyboardInterrupt 都接住，
   保证用户代码无论如何都不能让 runner “非正常退出而不回传结果”。
```

### 5.5 动作五：严格序列化输出

```python
try:
    blob = json.dumps(collected, ensure_ascii=False)   # 不加 default=str 兜底
except (TypeError, ValueError) as exc:
    emit({"ok": False, "error": {"code": "OUTPUT_NOT_SERIALIZABLE", ...}})
    return
if len(blob.encode("utf-8", "replace")) > MAX_OUTPUT_BYTES:
    emit({"ok": False, "error": {"code": "OUTPUT_TOO_LARGE", ...}})
    return
emit({"ok": True, "outputs": collected})
```

这里有个刻意的设计选择——**结果序列化不做 `default=str` 兜底**（注释明说）：

```text
如果用户 sink 了一个不可 JSON 化的对象（比如 datetime），
不悄悄把它转成字符串，而是显式报 OUTPUT_NOT_SERIALIZABLE，
让 Agent 得到明确信号（需要 datetime 请自己 .isoformat()）。
```

对比 `emit` 里写 logs 时用的是 `default=str`——**日志可以糊弄，结果不能糊弄**。这个不对称体现了“结果通道要严格、日志通道可宽松”的取舍。

### 5.6 动作六：限流

两条大小上限贯穿始终：

```python
MAX_LOG_BYTES = 64 * 1024       # 日志 64KB，_truncate 截断
MAX_OUTPUT_BYTES = 256 * 1024   # 输出 256KB，超了报 OUTPUT_TOO_LARGE
```

防止用户代码 `print` 一个 G 的日志、或 `sink` 一个巨大数组，把 sidecar 的内存或 Go↔sidecar 的管道刷爆。

---

## 6. Go↔Python 契约守卫（M5 反思③）

这是 M5 一个很有工程价值的设计，专门解决“两个语言各写一份 envelope，改一边忘了改另一边”的隐患。

### 6.1 问题：静默的字段错配

Go 侧 `code.go` 有一份 `codeRunResponse` 结构，Python 侧 `runner.py`/`main.py` 有一份 envelope。它们靠肉眼对齐。风险是（`contract_test.go` 注释）：

```text
任一边改字段名，另一边会静默解析出零值而不报错。
```

比如 Python 把 `outputs` 改成 `output`，Go 的 `json.Unmarshal` 不会报错，只会把 `Outputs` 解成空 map——**bug 是静默的**。

### 6.2 解法：一份 golden fixture，两侧都测它

`sidecar/contract/code_run.golden.json` 是唯一真相源：

```json
{
  "success_response": { "ok": true, "data": { "outputs": { "answer": 42 }, "logs": "debug line\n" } },
  "error_response":   { "ok": false, "error": { "code": "RUNTIME_ERROR", "message": "ValueError: boom" } }
}
```

两侧各写一个测试来锁它：

```text
Go 侧 contract_test.go：
  用 DisallowUnknownFields 严格解析 golden -> codeRunResponse
  golden 里有 Go 没声明的字段 -> 报错（提示 Go 要跟进）
  Go 结构接不住 golden 的字段 -> 报错

Python 侧 contract/test_contract.py：
  真跑 runner.py，验证它产出的字段名与 golden 一致
```

于是形成双向锁：

```text
任一边改字段名 -> 那一边的测试立刻失败 -> 静默 bug 变成显式 CI 失败
```

`contract_test.go` 用的 `strictUnmarshal` 是关键：

```go
func strictUnmarshal(data []byte, v any) error {
    dec := json.NewDecoder(bytes.NewReader(data))
    dec.DisallowUnknownFields()   // golden 多一个字段就报错
    return dec.Decode(v)
}
```

普通 `json.Unmarshal` 会忽略未知字段，而 `DisallowUnknownFields` 把“契约漂移”变成硬错误。这是跨语言契约测试的标准手法。

---

## 7. code 节点执行器：code.go（执行视角）

源码：`internal/engine/nodes/code.go`（M4 已介绍过结构，这里补 M5 视角）

它就是 M2/M4 讲的 `NodeExecutor` 接口的一个实现，干四件事：

```text
1. 从 nodeParam 取 code（空则报 "missing code"）和 timeout（缺省 30s）
2. 拼 codeRunRequest{code, inputs, timeout_sec}，POST 到 sidecar
3. 解析 codeRunResponse
4. ok=true -> NodeResult{Outputs}；ok=false -> Go error（带 code+message）
```

一个 M5 反思①②落地的细节——**sidecar 地址三级兜底**：

```go
func (e CodeExecutor) baseURL() string {
    if e.SidecarURL != "" {            // ① 优先用注入的（来自 config.Sidecar.BaseURL）
        return e.SidecarURL
    }
    if env := os.Getenv("SWF_SIDECAR_BASEURL"); env != "" {  // ② 环境变量
        return env
    }
    return "http://127.0.0.1:8090"     // ③ 本地默认
}
```

M4 讲的 `engine.New(store, sidecarURL)` 就是把 config 里的地址一路注入到这里。M5 反思②强调“config 打通，不再只靠环境变量兜底”——即优先级 ① 的引入。

还有一个超时配合的小心思：

```go
client = &http.Client{Timeout: time.Duration(timeoutSec+10) * time.Second}
```

Go 客户端超时 = 代码超时 + 10s。为什么多给 10s？留给 sidecar 收尾和回传的余量——避免“子进程刚好在超时边界跑完，结果还没传回 Go，Go 这边先超时断连”。

---

## 8. M5 测试怎么读？

M5 测试横跨两个语言，四组，可分“不需要真 sidecar”和“需要真 sidecar”两类。

### 8.1 Go 侧单测（mock sidecar，纯内存）

源码：`internal/engine/nodes/code_test.go`

用 `httptest` 起一个假 sidecar，覆盖 code 执行器的所有分支：

```text
TestCodeExecutor_Success           成功：outputs 解析对 + 请求体带上 code/inputs
TestCodeExecutor_RuntimeError      sidecar 回 ok=false -> Go error 带 code+message
TestCodeExecutor_MissingCode       nodeParam 无 code -> "missing code"
TestCodeExecutor_SidecarUnreachable 连不上 sidecar -> "call sidecar" 错误
TestCodeExecutor_BadResponse       sidecar 回非 JSON -> "bad sidecar response"
TestCodeExecutor_Registered        注册表装配 + SidecarURL 真被注入（反思②回归守卫）
```

这六个覆盖了 code.go 的每条错误路径，不依赖任何 Python 环境。

```bash
go test ./internal/engine/nodes/ -run TestCodeExecutor -v
```

### 8.2 Go 侧契约测试（读 golden，纯内存）

源码：`internal/engine/nodes/contract_test.go`

`TestContract_CodeRunResponse`：严格解析 golden，锁字段结构（见第 6 节）。也不需要 Python。

### 8.3 Python 侧测试（纯标准库，无需 pytest / 无需装依赖）

两个都用 `subprocess` 真跑 `runner.py`：

```text
contract/test_contract.py     Python 侧契约守卫：runner 产出字段与 golden 一致
test_runner_sanitize.py       TD-6 脱敏断言：
                                用例A 单帧异常 -> message 不含 runner.py / 绝对路径
                                用例B 多帧异常 -> message 只含 <user_code> 帧，
                                                 完整 traceback 留在 logs
```

运行（不用装 fastapi/pytest，因为 runner.py 只用标准库）：

```bash
python3 sidecar/contract/test_contract.py
python3 sidecar/test_runner_sanitize.py
```

`test_runner_sanitize.py` 特别值得看——它把 1.5 讲的脱敏行为固化成可判定断言：对外 message 泄露路径就失败，logs 里没有完整 traceback 也失败。这是“对外脱敏、对内可排障”两个要求同时上锁。

### 8.4 端到端集成测试（需真 sidecar）

源码：`internal/engine/nodes/code_integration_test.go`

```go
//go:build integration
func TestM5_CodeNodeRealSidecar(t *testing.T)
```

它验证 Go code 节点 → 真实 Python runner 的完整链路（子进程隔离 + sink 回传）：

```text
code: total = sum(inputs['nums']); print(...); sink({'total':.., 'n':..})
inputs: {nums: [1,2,3,4]}
断言: outputs.total == 10、outputs.n == 4
```

前置（注释里写明）：

```bash
# 先起 sidecar
sidecar/.venv/bin/python -m uvicorn main:app --host 127.0.0.1 --port 8090 --app-dir sidecar &
# 再跑
SWF_SIDECAR_BASEURL=http://127.0.0.1:8090 go test -tags integration ./internal/engine/nodes -run TestM5 -v
```

M4 讲义里 `TestRun_StartCodeToEnd` 也涉及 code 节点，但那个用 mock sidecar 测“调度器串起 code”；这个 M5 测试才真正打通到 Python 子进程。两者互补。

---

## 9. sidecar 的部署形态

源码：`deployments/Dockerfile.sidecar`

M5 也交付了 sidecar 的容器化，几个和运行安全相关的取舍值得记：

```dockerfile
FROM python:3.12-slim
RUN useradd -m -u 10001 swf          # 非 root 运行
...
COPY sidecar/requirements.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt   # 依赖层先装，命中缓存
COPY sidecar/main.py sidecar/runner.py ./
USER swf                             # 降权
EXPOSE 8090
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8090"]
```

三个关键点：

```text
1. --host 0.0.0.0（不是本地开发的 127.0.0.1）：否则 server 容器调
   http://sidecar:8090 会被拒（跨容器访问，见注释 pre-mortem R2）。
2. 非 root（USER swf）：即使用户代码逃逸出 Python 层，也只是个降权用户。
3. requirements 只有 3 个（fastapi/uvicorn/pydantic）：
   注意 runner.py 只用标准库，所以真正跑用户代码的子进程零第三方依赖，攻击面更小。
```

`runner.py` 只用标准库这点很重要：它意味着契约测试和脱敏测试能在任何有 Python3 的环境跑（不装 fastapi 也行），也意味着执行用户代码的那层没有第三方库的漏洞暴露面。

docker-compose 里 sidecar 是“五件套”之一（mysql/redis/server/worker/sidecar），一键起全套。

---

## 10. 当前 M5 的设计边界和后续增强

当前已经完成：

```text
1. FastAPI sidecar：/healthz + /run/python-code，统一 envelope
2. 子进程隔离执行：subprocess + 一次性 runner
3. 超时护栏：请求可申请更短、硬上限 60s、到点 kill
4. inputs 注入 + sink() 收集输出（对齐 PaiFlow 心智）
5. 输出隔离：抢占 fd1，print/stderr 进日志不污染结果
6. 分层错误码：SYNTAX/RUNTIME/OUTPUT_*/TIMEOUT/RUNNER_CRASHED...
7. TD-6 错误脱敏：对外裁绝对路径，完整栈留 logs
8. 输出限流：日志 64KB / 输出 256KB
9. 严格序列化：结果不做 default=str 兜底，不可序列化显式报错
10. Go↔Python 共享 golden 契约 + 两侧双向守卫
11. config 打通：sidecar 地址三级兜底，engine.New 注入
12. 容器化：非 root、绑 0.0.0.0、runner 零第三方依赖
```

当前还可以增强（部分是明确技术债）：

```text
1. TD-5 真沙箱：当前只是“进程隔离 + 超时”，挡不住 import os 读文件/发网络/
   吃满 CPU。生产级需 seccomp / gVisor / 每次执行独立容器 / 资源 cgroup 限制。
2. 依赖管理：用户代码只能用标准库，import 第三方库会 ModuleNotFoundError。
   若要支持，需预装白名单库或按需装（又扩大攻击面，需权衡）。
3. 并发与复用：当前每次请求 fork 一个新子进程，高频调用下 fork 开销可观。
   可考虑进程池 / 预热 worker（但要在“复用”和“状态隔离”间权衡）。
4. 输出大对象：256KB 上限之外，超大输出没有对象存储引用机制（M2 也提过 node_run 同类问题）。
5. 无 CPU/内存硬限：超时挡得住死循环，但挡不住“5 秒内吃 10G 内存”，需 cgroup/ulimit。
```

其中最值得强调的是 **TD-5**：M5 的隔离是“工程级”而非“安全级”。在受信任的内部评测场景够用，但若要跑真正不可信的外部代码，必须上真沙箱。这是 plan.md 明确挂账的技术债。

---

## 11. 一句话总结 M5

```text
M5 让工作流能安全地跑用户 Python 代码：Go 引擎不自己执行，而是通过 HTTP 把
code+inputs 交给常驻的 FastAPI sidecar，sidecar 再 fork 一个一次性子进程用
runner.py 来 exec——子进程抢占 fd1 把用户 print 隔离进日志、只让 sink() 的输出走
结果通道、异常按 SYNTAX/RUNTIME/OUTPUT_* 分类并脱敏（对外裁绝对路径、完整栈留 logs）、
日志 64KB 输出 256KB 限流、结果严格序列化不兜底；sidecar 负责超时 kill 与
TIMEOUT/RUNNER_CRASHED 归一；Go 与 Python 两侧靠一份 golden 契约双向锁字段名。
整套隔离是“进程隔离+超时”的工程护栏，非语言级真沙箱（TD-5），够用于受信任评测场景。
```
