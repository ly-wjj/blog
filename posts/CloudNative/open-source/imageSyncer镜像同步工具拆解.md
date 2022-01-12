​	 我们在使用云原生技术的时候，很可能会遇到镜像迁移的场景。面对此类问题，我们很容易想到，直接用docker的客户端pull，tag加push是不是就可以了？简单场景下，这样确实没什么问题，但是当我们需要迁移的镜像成千上万时，怎么办？弄个批量脚本，中间出错了怎么办？镜像太多，上传花费的时间太长怎么办？

那我这次隆重介绍一款镜像同步工具image-syncer。今天我就要拆解一下这个开源工具。开始拆解前，先简单介绍下这个开源工具的基本情况：

- 开源发起方：阿里巴巴
- start数：450+
- 开发语言：golang
- 代码仓库地址：https://github.com/AliyunContainerService/image-syncer
- 应用场景：镜像同步

## 功能特性

我们首先看看这款工具在github上介绍的特性：

1. 支持多对多镜像仓库同步

2. 支持基于Docker Registry V2搭建的docker镜像仓库服务 (如 Docker Hub、 Quay、 阿里云镜像服务ACR、 Harbor等)

3. 同步只经过内存和网络，不依赖磁盘存储，同步速度快

4. 增量同步, 通过对同步过的镜像blob信息落盘，不重复同步已同步的镜像

5. 并发同步，可以通过配置文件调整并发数

6. 自动重试失败的同步任务，可以解决大部分镜像同步中的网络抖动问题

7. 不依赖docker以及其他程序

### 特性分析

整个功能主要围绕易用，稳定，性能来描述。

- 易用：第1，2，7点，通过一定的配置，使用docker开源的client和registry，应该比较容易实现，就不多看了。
- 稳定：第6点，实现一个重试机制，这个应该是常规操作，貌似也解决了我前面提到的一些问题，可以稍微分析下。
- 性能：第3，4，5点，不经过磁盘，并发同步，这块应该是其核心要实现的地方，我们可以重点分析下。

## 使用方式

我们要拆解这个工具，首先我们先得弄清楚这个工具该怎么用咯。使用方式很简单，大概分为如下几步：

1. 配置仓库认证信息

   ```json
   {  
       // 认证字段，其中每个对象为一个registry的一个账号和
       // 密码；通常，同步源需要具有pull以及访问tags权限，
       // 同步目标需要拥有push以及创建仓库权限，如果没有提供，则默认匿名访问
       
       "quay.io": {    // 支持 "registry" 和 "registry/namespace"（v1.0.3之后的版本） 的形式，需要跟下面images中的registry(registry/namespace)对应
                       // images中被匹配到的的url会使用对应账号密码进行镜像同步, 优先匹配 "registry/namespace" 的形式
           "username": "xxx",               // 用户名，可选，（v1.3.1 之后支持）valuse 使用 "${env}" 或者 "$env" 类型的字符串可以引用环境变量
           "password": "xxxxxxxxx",         // 密码，可选，（v1.3.1 之后支持）valuse 使用 "${env}" 或者 "$env" 类型的字符串可以引用环境变量
           "insecure": true                 // registry是否是http服务，如果是，insecure 字段需要为true，默认是false，可选，支持这个选项需要image-syncer版本 > v1.0.1
       },
       "registry.cn-beijing.aliyuncs.com": {
           "username": "xxx",
           "password": "xxxxxxxxx"
       },
       "registry.hub.docker.com": {
           "username": "xxx",
           "password": "xxxxxxxxxx"
       },
       "quay.io/coreos": {                       
           "username": "abc",              
           "password": "xxxxxxxxx",
           "insecure": true  
       }
   }
   ```

2. 配置镜像同步配

   ```json
   {
       // 同步镜像规则字段，其中条规则包括一个源仓库（键）和一个目标仓库（值）
       
       // 同步的最大单位是仓库（repo），不支持通过一条规则同步整个namespace以及registry
       
       // 源仓库和目标仓库的格式与docker pull/push命令使用的镜像url类似（registry/namespace/repository:tag）
       // 源仓库和目标仓库（如果目标仓库不为空字符串）都至少包含registry/namespace/repository
       // 源仓库字段不能为空，如果需要将一个源仓库同步到多个目标仓库需要配置多条规则
       // 目标仓库名可以和源仓库名不同（tag也可以不同），此时同步功能类似于：docker pull + docker tag + docker push
   
       "quay.io/coreos/kube-rbac-proxy": "quay.io/ruohe/kube-rbac-proxy",
       "xxxx":"xxxxx",
       "xxx/xxx/xx:tag1,tag2,tag3":"xxx/xxx/xx"
   
       // 当源仓库字段中不包含tag时，表示将该仓库所有tag同步到目标仓库，此时目标仓库不能包含tag
       // 当源仓库字段中包含tag时，表示只同步源仓库中的一个tag到目标仓库，如果目标仓库中不包含tag，则默认使用源tag
       // 源仓库字段中的tag可以同时包含多个（比如"a/b/c:1,2,3"），tag之间通过","隔开，此时目标仓库不能包含tag，并且默认使用原来的tag
       
       // 当目标仓库为空字符串时，会将源镜像同步到默认registry的默认namespace下，并且repo以及tag与源仓库相同，默认registry和默认namespace可以通过命令行参数以及环境变量配置，参考下面的描述
   }
   ```

3. 启动工具

   ```shell
   # 设置配置文件为config.json，默认registry为registry.cn-beijing.aliyuncs.com
   # 默认namespace为ruohe，并发数为6
   ./image-syncer --proc=6 --auth=./auth.json --images=./images.json --namespace=ruohe \
   --registry=registry.cn-beijing.aliyuncs.com --retries=3
   ```

   

整个过程和配置还是比较简单的，详细信息大家可以看下文档：https://github.com/AliyunContainerService/image-syncer/blob/master/README-zh_CN.md

## 核心功能分析

该开源工具主要就是镜像同步，那我们就直击核心功能。

### 镜像同步

镜像同步功能，我的理解它的核心要去解决如下几个问题：

1. 同步不经过磁盘
2. 已同步过的不能重复同步
3. 多并发同步

分析这几点前，我们首先需求了解镜像原理和镜像获取和推送的实现原理。

#### 镜像原理

我们可能都了解镜像元数据管理的相关概念repository、image、layer。

- repository即由具有某个功能Docker镜像的所有迭代版本构成的镜像库
- image元数据包括了镜像架构，操作系统，镜像默认配置，构建该镜像的容器ID和配置，创建时间，创建该镜像的Docker版本，构建镜像的历史信息以及rootfs组成。
- layer对应的是镜像层的概念。

Layer这个概念想多复杂一点，该工具的一个实现就要在这个上面做文章，我这里就多啰嗦几句。

#### layer

layer是真正与物理镜像文件想对应的。layer主要分为两种，其分别对应代码中的两种接口

- 只读层，Layer。实际通过roLayer来实现该接口。
- 读写层，RWLayer。实际通过mountedLayer来实现该接口。

##### roLayer

其元数据信息的持久化文件位于/var/lib/docker/image/[graph_driver]/layerdb/sha256/文件夹下。roLayer主要存储的信息是：

- 镜像层chainID
- 校验码diffID
- 父镜像层parent
- cacheID，基于存储驱动存储当前镜像层文件对应的ID
- 该镜像层的大小size

镜像层的这些信息又是怎么来得呢？diffID和size是可以通过计算得到；chainID和父镜像层parent需要从所属image元数据中计算得到；而cacheID是在当前Docker宿主机上随机生成的uuid，在当前宿主机上与该镜像层一一对应，用于标示并索引存储驱动中的镜像层文件。

​	在layer的所有属性中，diffID采用SHA256算法，基于镜像层文件包的内容计算得到。chainID是基于内容存储的索引，它是根据当前层与所有祖先镜像层diffID计算出来的，具体算法如下：

- 如果该镜像层是最底层（没有父镜像层），该层的diffID便是chainID。
- 该镜像层的chainID计算公式为chainID(n)=SHA256(chain(n-1) diffID(n))，也就是根据父镜像层的chainID加上一个空格和当前层的diffID，再计算SHA256校验码。

##### mountedLayer

​	mountedLayer存储的内容主要为索引某个容器的可读写层（也叫容器层）的ID（也对应容器的ID）、容器的init层在存储驱动（graphdriver）的ID为initID、读写层在存储驱动中的ID为mountID以及容器层的父层镜像的chainID，即parent。持久化文件位于/var/lib/docker/image/[graph_driver]/layerdb/mounts/[container_id]/路径下。

#### 镜像获取和推送原理

一般情况下，我们会选择通过docker的客户端的命令行方式即可实现镜像的同步，即

```shell
## 拉取源镜像仓库镜像
docker pull src/image_name:tag
## 重新对镜像打标签
docker tag src/xxximage_nametag dest/image_name:tag
## 推送镜像到目标仓库
docker push dest/image_name:tag
```

而在获取镜像和推送镜像都离不开两个概念，一个是获取mainfest，一个是获取layer数据。我们通过获取镜像的方式来简单了解下其原理。

1. 获取mainfest，mainfest包含镜像的相关元数据信息。其mainfest的格式如下：

   ```json
   digest: sha256:6d47a9873783f7bf23773f0cf60c67cef295d451f56b8b79fe3a1ea217a4bf98
   payload: 
   {
      "schemaVersion": 2,
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "config": {
         "mediaType": "application/vnd.docker.container.image.v1+json",
         "size": 3688,
         "digest": "sha256:790ca1071242c78a2ced2984954322d94e9d2b94829e1839804af5340087d6c7"
      },
   		"layers": [
         {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 1991747,
            "digest": "sha256:605ce1bd3f3164f2949a30501cc596f52a72de05da1306ab360055f0d7130c32"
         },
         {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 1843790,
            "digest": "sha256:77a50f74e4304f93f39148b88dbe5f83400ab120d04856894db4be294f47bf7d"
         },
         {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 7738438,
            "digest": "sha256:7118e0a5b59500ceeb4c9686c952cae5e8bfe3e64e5afaf956c7098854a2390e"
         },
         {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 1474,
            "digest": "sha256:7ee27521e3b23c3c2713acb1394e45a601ae894bbb5d4bf1761ebce8e97060a3"
         },
         {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 6329,
            "digest": "sha256:71b1ca055a6770e6e61e1573719dd450982998f9e483e02e27ab047d79929a2e"
         },
         {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 1054564,
            "digest": "sha256:e0deed02c7d20c528f44000b96ec2c3269307de184ece214a7ff3c8e44f3d16d"
         },
         {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 520,
            "digest": "sha256:6d140169e8a78b952428f4c20a9ab7344b6efadcf67af25a45c93106ecfa8e65"
         }
      ]
   }
   ```

2. 我们通过mainfest，逐个获取对应的镜像的layer的内容数据。

### 功能实现

基于前面介绍的镜像相关原理之后，我们基本可以猜测该功能可以按如下方式去实现：

1. 通过镜像仓库获取mainfest
2. 通过mainfest获取到镜像对应的镜像层信息
3. 逐步获取各个镜像层的文件内容，不通过存储驱动保存到本地，而是重新将镜像层推送到远程仓库

这样就可以省略掉存储驱动将文件内容存储到本地的步骤，减少了磁盘IO，可以大大提供镜像同步的速度。

为了增量上传，我们可以按如下方式实现：

1. 上传远程仓库镜像成功保存一份信息在本地
2. 每次上传镜像前，判断是否上传过该镜像，如果已上传则跳过。

并发上传可以利用golang自身语言的特性，创建一个goroutine任务池，用于镜像同步的任务执行。

## 代码拆解

基于我们猜测的一个实现过程，带着这个思路去拆解，我们对image-syncer的代码。先看下我们的核心Run()的代码。

```go
// Run is main function of a synchronization client
func (c *Client) Run() {
	fmt.Println("Start to generate sync tasks, please wait ...")

	//var finishChan = make(chan struct{}, c.routineNum)

	// open num of goroutines and wait c for close
	openRoutinesGenTaskAndWaitForFinish := func() {
		wg := sync2.WaitGroup{}
		for i := 0; i < c.routineNum; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()
				for {
					urlPair, empty := c.GetAURLPair()
					// no more task to generate
					if empty {
						break
					}
					moreURLPairs, err := c.GenerateSyncTask(urlPair.source, urlPair.destination)
					if err != nil {
						c.logger.Errorf("Generate sync task %s to %s error: %v", urlPair.source, urlPair.destination, err)
						// put to failedTaskGenerateList
						c.PutAFailedURLPair(urlPair)
					}
					if moreURLPairs != nil {
						c.PutURLPairs(moreURLPairs)
					}
				}
			}()
		}
		wg.Wait()
	}

	openRoutinesHandleTaskAndWaitForFinish := func() {
		wg := sync2.WaitGroup{}
		for i := 0; i < c.routineNum; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()
				for {
					task, empty := c.GetATask()
					// no more tasks need to handle
					if empty {
						break
					}
					if err := task.Run(); err != nil {
						// put to failedTaskList
						c.PutAFailedTask(task)
					}
				}
			}()
		}

		wg.Wait()
	}

	for source, dest := range c.config.GetImageList() {
		c.urlPairList.PushBack(&URLPair{
			source:      source,
			destination: dest,
		})
	}

	// generate sync tasks
	openRoutinesGenTaskAndWaitForFinish()

	fmt.Println("Start to handle sync tasks, please wait ...")

	// generate goroutines to handle sync tasks
	openRoutinesHandleTaskAndWaitForFinish()

	for times := 0; times < c.retries; times++ {
		if c.failedTaskGenerateList.Len() != 0 {
			c.urlPairList.PushBackList(c.failedTaskGenerateList)
			c.failedTaskGenerateList.Init()
			// retry to generate task
			fmt.Println("Start to retry to generate sync tasks, please wait ...")
			openRoutinesGenTaskAndWaitForFinish()
		}

		if c.failedTaskList.Len() != 0 {
			c.taskList.PushBackList(c.failedTaskList)
			c.failedTaskList.Init()
		}

		if c.taskList.Len() != 0 {
			// retry to handle task
			fmt.Println("Start to retry sync tasks, please wait ...")
			openRoutinesHandleTaskAndWaitForFinish()
		}
	}
	...
}
```

核心逻辑如下

1. 根据配置生成urlPairList
2. 生成同步任务列表
3. 生成各个执行任务的goroutine开始执行任务

这段代码其实包含了上面提到的几块内容

- 并发实现
- 错误重试

我们就此可以继续深挖下

### 并发实现

wg := sync2.WaitGroup{} ，这里是利用golang的sync包，sync包提供了共享内存，锁，等机制进行协同操作的包。这里利用锁的方式来实现并发，是否是好的实现呢？

就目前的场景来看，我们需要先快速生成执行任务的列表清单，再执行清单中的任务。一个 goroutine 需要等待一批 goroutine 执行完毕以后才继续执行，那么这种多线程等待的问题是可以使用 WaitGroup 的。这么看来问题也不大。

### 错误重试

这里我们看到重试实现

1. 利用for循环来实现重试的次数
2. 利用一个failedTaskGenerateList来保存生成任务列表失败的情况
3. 将failedTaskGenerateList中的任务放回待执行urlPairList中，并清空failedTaskGenerateList
4. 执行openRoutinesGenTaskAndWaitForFinish，再次生成任务清单
5. 利用failedTaskList判断是否存在执行错误的任务
6. 执行openRoutinesHandleTaskAndWaitForFinish，再次执行同步任务

这里我觉得这种实现有点简单粗暴，作为一个兜底重试保留也可以。如果需要更好的实现，我觉得任务的执行失败应该在任务侧做失败重试会比较更加合理。

### 同步镜像

这个任务执行的方法代码有点长了，我尽量将重点摘了出来，大家将就看下，后面以此作为反面教材吧。

```go
// Run is the main function of a sync task
func (t *Task) Run() error {
	// get manifest from source
	manifestBytes, manifestType, err := t.source.GetManifest()
	...
	manifestInfoSlice, thisManifestInfo, err := ManifestHandler(manifestBytes, manifestType,
		t.osFilterList, t.archFilterList, t.source, nil)
	...
	if len(manifestInfoSlice) == 0 {
		...
		return nil
	}
	blobInfos, err := t.source.GetBlobInfos(manifestInfoSlice)
	...

	// blob transformation
	for _, b := range blobInfos {
		blobExist, err := t.destination.CheckBlobExist(b)
		...
		if !blobExist {
			// pull a blob from source
			blob, size, err := t.source.GetABlob(b)
			...
			b.Size = size
			// push a blob to destination
			...
		} else {
			...
		}

	}
	// Push manifest list
	if manifestType == manifest.DockerV2ListMediaType {
		var manifestSchemaListInfo *manifest.Schema2List
		if thisManifestInfo == nil {
			manifestSchemaListInfo, err = manifest.Schema2ListFromManifest(manifestBytes)
		} else {
			manifestSchemaListInfo = thisManifestInfo.(*manifest.Schema2List)
			manifestBytes, err = manifestSchemaListInfo.Serialize()
		}
		...
		var subManifestByte []byte

		// push manifest to destination
		for _, manifestDescriptorElem := range manifestSchemaListInfo.Manifests {
			...
			subManifestByte, _, err = t.source.source.GetManifest(t.source.ctx, &manifestDescriptorElem.Digest)
			...
			if err := t.destination.PushManifest(subManifestByte); err != nil {
				...
			}
			...
		}

		// push manifest list to destination
		if len(manifestInfoSlice) != 0 {
			if err := t.destination.PushManifest(manifestBytes); err != nil {
				...
			}
			...
	} else if len(manifestInfoSlice) != 0 {
		// push manifest to destination
		if err := t.destination.PushManifest(manifestBytes); err != nil {
			...
		}
		...
	}
	...
	return nil
}
```

核心逻辑咱们也理一下：

1. t.source.GetManifest()后去源的mainfest
2. t.source.GetBlobInfos(manifestInfoSlice)获取源的blob信息
3. 遍历blobinfos
   1. t.destination.CheckBlobExist(b)检查目的仓库是否存在
   2. 如果目的仓库不存在该blob，t.source.GetABlob(b)获取源仓库的blob
   3. t.destination.PutABlob(blob, b)将blob信息推送到目的仓库
4. 如果mainfestType为DockerV2ListMediaType，遍历manifestSchemaListInfo.Manifests
   1. t.source.source.GetManifest(t.source.ctx, &manifestDescriptorElem.Digest)获取源仓库mainfest。
   2. t.destination.PushManifest(subManifestByte) 推送mainfest到目的仓库。
   3. 判断manifestInfoSlice的切片长度不为0，t.destination.PushManifest(manifestBytes)推送mainfest到远程仓库
5. 判断manifestInfoSlice的切片长度不为0，t.destination.PushManifest(manifestBytes)推送mainfest到远程仓库。（这里我感觉要重复逻辑，可以优化下）

### 增量同步

这里官方的功能描述里其实是有错误的，在1.1.0版本后就已经移除了迁移记录落盘的方式。详情见https://github.com/AliyunContainerService/image-syncer/commit/27dc0644afb3c8ce69cb37d7fd829ba7a283cbec

既然不需要落盘了，那这段逻辑又是怎么实现的呢？我们可以看下这个方法：

```go
func (i *ImageDestination) CheckBlobExist(blobInfo types.BlobInfo) (bool, error) {
	exist, _, err := i.destination.TryReusingBlob(i.ctx, types.BlobInfo{
		Digest: blobInfo.Digest,
		Size:   blobInfo.Size,
	}, NoCache, false)

	return exist, err
}
```

这里其实在前面也提到过这个方法，通过去远程仓库判断是否存在有需要上传的blob。而其中TryReusingBlob是官方提供的接口，这样用起来也挺香的。

## 拆解总结

   我从两方面来总结这个镜像同步的开源工具吧。咱们先说好的方面：

1. 基本的功能还是能够满足同步要求的，易用性方面相比使用docker客户端的方式就不用多少，方便太多了。
2. 同步效率方面也确实提升了很多。这个自己亲测过哟！
3. 文档上还算比较完整，基本上按照文档操作，就能开始搬砖。

再谈谈拆解过程中遇到一些觉得可以优化的地方：

1. 代码书写格式可以要求高一点。
2. 为什么重试不放在每个任务上去实现，这样可以省去再次生成任务清单的逻辑，这里个有个大大的疑问。
3. Mainfest的转换这块代码逻辑有点冗余，docker客户端的开源代码里应该有现成的方法，是否可以像TryReusingBlob那样使用其阿里。

最后还是要致谢所有开源者，为我们提供免费好用的工具！像我这种"鸡蛋里挑骨头"的人，有本事自己去做点开源贡献呀！我这篇文章也算是做一点贡献吧。