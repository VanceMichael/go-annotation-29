# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

结算轮询超时之后报成功，可是一轮都没跑完，下游拿这个结果当已结算处理了。

```
$ ./dramactl settle poll --timeout 120ms --poll-latency 5s
{
  "elapsed_ms": 120,
  "ok": true,
  "rounds": 0,
  "settled": false,
  "state": "pending",
  "stats": {
    "submits": 1,
    "polls": 0
  },
  "timeout_ms": 120,
  "want_rounds": 3
}
$ echo $?
0
```

这一份输出自己就是矛盾的：`ok` 是 true、退出码 0，但 `rounds` 是 0、`settled` 是 false、`state` 还停在 pending。按 README，超时或取消时必须返回可沿错误链判定的 context 错误，并且不得把中止的轮询当作已结算终态上报 —— 期望是退出码 4。

调用方侧的表现是 `errors.Is(err, context.DeadlineExceeded)` 永远不成立，因为根本没拿到 error。

几个对照观察：

- `--timeout 3s --poll-latency 10ms`（超时足够）正常走完：`rounds` 3、`state` 是 settled、`settled` 是 true、退出码 0。
- 出问题的那次里 `submits` 是 1、`polls` 是 0，说明提交这一步已经正常完成，卡住的是轮询。
- `elapsed_ms` 恰好等于 120，说明它确实是在超时那一刻返回的，不是等满 5 秒。

请先不要修改代码。先帮我定位根因，讲清楚为什么超时中止会被当成正常返回、为什么提交阶段的超时表现反而是正常的，并给出实际执行过的复现命令与观察到的输出作为证据。结论确认后再讨论怎么改。

## 含 Bug 版本

- 仓库：VanceMichael/go-annotation-29
- 仓库地址：https://github.com/VanceMichael/go-annotation-29.git
- parent SHA：c673fd662df1bff3fbd3658d70f9d705ed2a25eb

## 复现步骤

```bash
git clone -- https://github.com/VanceMichael/go-annotation-29.git bug-repro
cd bug-repro
git checkout --detach c673fd662df1bff3fbd3658d70f9d705ed2a25eb
go test ./internal/gateway/ -run "TestPollSurfacesContextError|TestPollDoesNotReportSuccessOnTimeout|TestPollHonoursCancel|TestPollAlreadyExpiredContext|TestSubmitAndPollSurfacesPollTimeout" -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/gateway/ -run "TestPollSurfacesContextError|TestPollDoesNotReportSuccessOnTimeout|TestPollHonoursCancel|TestPollAlreadyExpiredContext|TestSubmitAndPollSurfacesPollTimeout" -count=1
--- FAIL: TestPollSurfacesContextError (0.04s)
    gateway_test.go:26: 调用方超时后轮询应返回错误
--- FAIL: TestPollDoesNotReportSuccessOnTimeout (0.03s)
    gateway_test.go:46: 超时后返回了 nil 错误, 结果 = {TaskID:T-0002 State:pending AmountFen:0 Rounds:0}
--- FAIL: TestPollHonoursCancel (0.02s)
    gateway_test.go:69: 取消后应返回 context.Canceled, 实际 <nil>
--- FAIL: TestPollAlreadyExpiredContext (0.00s)
    gateway_test.go:87: 已取消的 context 应返回 context.Canceled, 实际 <nil>
--- FAIL: TestSubmitAndPollSurfacesPollTimeout (0.04s)
    gateway_test.go:213: 轮询超时应返回错误, 结果 = {TaskID:T-0012 State:pending AmountFen:1000 Rounds:0}
FAIL
FAIL	microdrama/internal/gateway	0.169s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/gateway/ -run "TestPollSurfacesContextError|TestPollDoesNotReportSuccessOnTimeout|TestPollHonoursCancel|TestPollAlreadyExpiredContext|TestSubmitAndPollSurfacesPollTimeout" -count=1
--- FAIL: TestPollSurfacesContextError (0.04s)
    gateway_test.go:26: 调用方超时后轮询应返回错误
--- FAIL: TestPollDoesNotReportSuccessOnTimeout (0.03s)
    gateway_test.go:46: 超时后返回了 nil 错误, 结果 = {TaskID:T-0002 State:pending AmountFen:0 Rounds:0}
--- FAIL: TestPollHonoursCancel (0.02s)
    gateway_test.go:69: 取消后应返回 context.Canceled, 实际 <nil>
--- FAIL: TestPollAlreadyExpiredContext (0.00s)
    gateway_test.go:87: 已取消的 context 应返回 context.Canceled, 实际 <nil>
--- FAIL: TestSubmitAndPollSurfacesPollTimeout (0.04s)
    gateway_test.go:213: 轮询超时应返回错误, 结果 = {TaskID:T-0012 State:pending AmountFen:1000 Rounds:0}
FAIL
FAIL	microdrama/internal/gateway	0.136s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

目标仓库零改动（git status 干净，无新增、修改或删除文件）。
准确指出出问题的 Go 文件与具体符号。
说明轮询在调用方 context 结束时为什么返回了 nil 错误与当前进度，并解释这一点如何让调用方按成功路径继续、使输出同时出现「ok 为 true」与「rounds 为 0、state 仍是 pending」这对自相矛盾的字段，退出码停在 0。
解释为什么提交阶段的超时处理不受影响（submits 为 1、polls 为 0），从而说明症状只出现在轮询这条路径上。
给出实际执行过的复现命令与观察到的输出作为证据，而非仅凭阅读代码推断。
