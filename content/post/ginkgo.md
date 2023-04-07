---
title: "Ginkgo源码"
date: 2023-04-05T08:56:03+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: []
tags: []
---

# ginkgo cmd

执行ginkgo cmd默认运行的是ginkgo run

{{< figure src="/ginkgo/Untitled.png" width="700px" >}}

入口函数

```go
func (r *SpecRunner) RunSpecs(args []string, additionalArgs []string) {
```

主要做两件事，编译和运行suite

## 编译

{{< figure src="/ginkgo/Untitled 1.png" width="700px" >}}

使用 `go test -c -o` 得到`.test`可执行文件

{{< figure src="/ginkgo/gotest.png" width="700px" >}}

要理解.test文件中的测试用例（spec）是怎么执行的，就要理解ginkgo是怎么构造specs tree的

### 构造specs tree

`Describe`等是container Node，`BeforeXXX`和`AfterXXX`等是setup Node，`It`是subject Node。`Describe`通过`var _ = Describe`的方式实现在golang源文件顶层执行函数，这会使得在编译时最先执行Describe。

他们底层都是通过`pushNode`，从外向内一层一层的push Node，然后在`RunSpecs`入口中的`BuildTree`构造出如下的specs tree数据结构：

{{< figure src="/ginkgo/image3.png" width="1000px" >}}

## 运行

### ginkgo

在SUITE_LOOP中循环运行每一个编译出来的`.test`：

```go
suites[suiteIdx] = internal.RunCompiledSuite(suites[suiteIdx], r.suiteConfig, r.reporterConfig, r.cliConfig, r.goFlagsConfig, additionalArgs)
```

每个suite都会编译成一个`.test`，如果有多个test suite，一个一个的编译运行

如果以parallel的模式运行，则启动server

```go
server, err := parallel_support.NewServer(numProcs, reporters.NewDefaultReporter(reporterConfig, formatter.ColorableStdOut))
```

生成go test flag，如果运行在parallel模式，启动多个.test进程并发执行测试用例。ginkgo会根据机器的核心数决定启动多少个test进程，每个test进程运行的都是同一组specs，但是test具体运行哪一个spec由ginkgo server决定。

{{< figure src="/ginkgo/Untitled 2.png" width="1000px" >}}

启动goroutine等待子进程退出并卡住等待结果

```go
go func() {
	cmd.Wait()
	exitStatus := cmd.ProcessState.Sys().(syscall.WaitStatus).ExitStatus()
	procResults <- procResult{
		passed:               (exitStatus == 0) || (exitStatus == types.GINKGO_FOCUS_EXIT_CODE),
		hasProgrammaticFocus: exitStatus == types.GINKGO_FOCUS_EXIT_CODE,
	}
}()

// 然后卡住等待子进程结果
passed := true
for proc := 1; proc <= cliConfig.ComputedProcs(); proc++ {
	result := <-procResults
	passed = passed && result.passed
	suite.HasProgrammaticFocus = suite.HasProgrammaticFocus || result.hasProgrammaticFocus
}
```

### .test

.test文件的入口：

```go
func TestE2E(t *testing.T) {
	gomega.RegisterFailHandler(ginkgo.Fail)
	ginkgo.RunSpecs(t, "Sample Suite")
}
```

如果运行在parallel模式，则启动client并连接server

```go
client = parallel_support.NewClient(suiteConfig.ParallelHost)
if !client.Connect() {
	client = nil
	exitIfErr(types.GinkgoErrors.UnreachableParallelHost(suiteConfig.ParallelHost))
}
```

Build specs tree

```go
err := global.Suite.BuildTree()
```

Run Suite

1. 把specs随机打散分组
    
    > ordered specs分在一组，普通的spec单独一个是一组，serial specs分为一组
    > 
    
    ```go
    groupedSpecIndices, serialGroupedSpecIndices := OrderSpecs(specs, suite.config)
    ```
    
2. 以group为单位runSpecs。具体下一步运行哪一组从server获得下标
    
    ```go
    nextIndex = suite.client.FetchNextCounter
    ```
    
3. 如果是serial group，那么必须当前是#1 process，而且要等到其他process都退出。
    
    ```go
    if suite.config.ParallelProcess == 1 && len(serialGroupedSpecIndices) > 0 {
    	groupedSpecIndices, serialGroupedSpecIndices, nextIndex = serialGroupedSpecIndices, GroupedSpecIndices{}, MakeIncrementingIndexCounter()
    	suite.client.BlockUntilNonprimaryProcsHaveFinished()
    	continue
    }
    ```
    
4. 开始Run group specs
    
    每一组里可能多个specs，一个spec一个spc的执行
    
    runSpec
    
    1. 判断interruptStatus，来决定要不要skip 这个spec
    2. 根据attempt derocator的定义，尝试attempts次
    3. spec中的nodes分为两批执行，先把spec中的setup Node和subject Node（It）组成为一批nodes依次运行
        
        ```go
        nodes := spec.Nodes.WithType(types.NodeTypeBeforeAll)
        nodes = append(nodes, spec.Nodes.WithType(types.NodeTypeBeforeEach)...).SortedByAscendingNestingLevel()
        nodes = append(nodes, spec.Nodes.WithType(types.NodeTypeJustBeforeEach).SortedByAscendingNestingLevel()...)
        nodes = append(nodes, spec.Nodes.FirstNodeWithType(types.NodeTypeIt))
        ```
        
    4. 再把cleanup Node组成一批Nodes依次运行。
        
        ```go
        nodes := spec.Nodes.WithType(types.NodeTypeAfterEach)
        nodes = append(nodes, spec.Nodes.WithType(types.NodeTypeAfterAll)...).SortedByDescendingNestingLevel()
        nodes = append(spec.Nodes.WithType(types.NodeTypeJustAfterEach).SortedByDescendingNestingLevel(), nodes...)
        ```
        
    5. nodes中的Node一个一个的执行，runNode
        
        runNode
        
        1. 判断interruptStatus，来决定Node要不要跳过，但是cleanup、report的Node必须要收到多次信号才会真正跳过。
        2. It中用户定义的closure在底层就是通过goroutine执行：
            
            ```go
            go func() {
            	finished := false
            	defer func() {
            		if e := recover(); e != nil || !finished {
            			suite.failer.Panic(types.NewCodeLocationWithStackTrace(2), e)
            		}
            
            		outcomeFromRun, failureFromRun := suite.failer.Drain()
            		failureFromRun.TimelineLocation = suite.generateTimelineLocation()
            		outcomeC <- outcomeFromRun
            		failureC <- failureFromRun
            	}()
            
            	// It 中定义的closure
            	node.Body(sc)
            	finished = true
            }()
            ```
            
        3. 这个goroutine有几个退出条件：
            
            {{< figure src="/ginkgo/Untitled 3.png" width="800px" >}}
            
            超时和interrupt的情况下都会再等待gracePeriod，但是如果Node没定义context，则gracePeriod为0
            
            ```go
            if !node.HasContext {
            	gracePeriod = 0
            }
            ```
            
            如果gracePeriod时间到了Node还没有执行完，那就leak Node，也就是leak goroutine，这可能会导致无法预测的行为。
            
            **interrupt机制：**
            
            这个channel是从interrupt handler复制过来的，有几种interrupt的情况
            
            - interrupt handler每隔500ms去服务端轮询是否可以Abort。
                
                什么时候可以Abort呢：如果运行在paralle模式，并且开启了fail-fast，这样有任何一个process失败，都会通知服务端现在可以开始Abort了
                
                {{< figure src="/ginkgo/Untitled 4.png" width="800px" >}}
                
                如果可以那么就会触发这个interrupt channel close。
                
                > 这个地方的实现是通过轮询来实现的，状态更新会有500ms的延迟。存在两个问题：
                > 1. 在运行Serial的Node或者cleanup Node时，会先检查一下状态，再决定是否运行。但是可能当时server端已经设置为需要abort了，可是还需要等500ms才能拿到实际状态。这个时候会去运行Node，但是实际上是应该skip的。
                > 2. cleanup Node应该直接忽略Abort的channel。
                > 
                > 我在提交了一个PR [https://github.com/onsi/ginkgo/pull/1178](https://github.com/onsi/ginkgo/pull/1178) 解决这个问题
                
            - 收到SIGINT和SIGTERM信号
                
                ```go
                func NewInterruptHandler(client parallel_support.Client, signals ...os.Signal) *InterruptHandler {
                	if len(signals) == 0 {
                		signals = []os.Signal{os.Interrupt, syscall.SIGTERM}
                	}
                ```
                

运行AfterSuiteCleanup Node
