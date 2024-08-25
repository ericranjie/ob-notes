# 
原创 theanarkh 编程杂技

 _2021年10月23日 01:41_

前言：在 Node.js 中，我们有时候需要抓取进程堆快照来判断是否有内存泄漏，本文介绍Node.js 中抓取堆快照的实现。

  

首先来看一下 Node.js 中如何抓取堆快照。

const { Session } = require('inspector');

const session = new Session();

let chunk = '';

const cb = (result) => {

  chunk += result.params.chunk;

};

  
session.on('HeapProfiler.addHeapSnapshotChunk', cb);  
session.post('HeapProfiler.takeHeapSnapshot', (err, r) => {  session.off('HeapProfiler.addHeapSnapshotChunk', cb);  

    console.log(err || chunk);

});

下面看一下 HeapProfiler.addHeapSnapshotChunk 命令的实现。

{      v8_crdtp::SpanFrom("takeHeapSnapshot"),      &DomainDispatcherImpl::takeHeapSnapshot  
}

对应 DomainDispatcherImpl::takeHeapSnapshot 函数。

void DomainDispatcherImpl::takeHeapSnapshot(const v8_crdtp::Dispatchable& dispatchable){    std::unique_ptr<DomainDispatcher::WeakPtr> weak = weakPtr();    // 抓取快照 DispatchResponse response = m_backend->takeHeapSnapshot(std::move(params.reportProgress), std::move(params.treatGlobalObjectsAsRoots), std::move(params.captureNumericValue));    // 抓取完毕，响应    if (weak->get())        weak->get()->sendResponse(dispatchable.CallId(), response);  

    return;

}

上面代码中 m_backend 是 V8HeapProfilerAgentImpl 对象。

Response V8HeapProfilerAgentImpl::takeHeapSnapshot(    Maybe<bool> reportProgress, Maybe<bool> treatGlobalObjectsAsRoots,    Maybe<bool> captureNumericValue) {  v8::HeapProfiler* profiler = m_isolate->GetHeapProfiler();  // 抓取快照  const v8::HeapSnapshot* snapshot = profiler->TakeHeapSnapshot(      progress.get(), &resolver, treatGlobalObjectsAsRoots.fromMaybe(true),      captureNumericValue.fromMaybe(false));  // 抓取完毕后通知调用方    HeapSnapshotOutputStream stream(&m_frontend);  snapshot->Serialize(&stream);  const_cast<v8::HeapSnapshot*>(snapshot)->Delete();  // HeapProfiler.takeHeapSnapshot 命令结束，回调调用方  

  return Response::Success();

}

我们重点看一下 profiler->TakeHeapSnapshot。

const HeapSnapshot* HeapProfiler::TakeHeapSnapshot(    ActivityControl* control, ObjectNameResolver* resolver,    bool treat_global_objects_as_roots, bool capture_numeric_value) {  return reinterpret_cast<const HeapSnapshot*>(      reinterpret_cast<i::HeapProfiler*>(this)->TakeSnapshot(          control, resolver, treat_global_objects_as_roots,  

          capture_numeric_value));

}

继续看真正的 TakeSnapshot。

HeapSnapshot* HeapProfiler::TakeSnapshot(    v8::ActivityControl* control,    v8::HeapProfiler::ObjectNameResolver* resolver,    bool treat_global_objects_as_roots, bool capture_numeric_value) {  is_taking_snapshot_ = true;  HeapSnapshot* result = new HeapSnapshot(this, treat_global_objects_as_roots,                                          capture_numeric_value);  {    HeapSnapshotGenerator generator(result, control, resolver, heap());    if (!generator.GenerateSnapshot()) {      delete result;      result = nullptr;    } else {      snapshots_.emplace_back(result);    }  }  

  return result;

}

我们看到新建了一个 HeapSnapshot 对象，然后通过 HeapSnapshotGenerator 对象的 GenerateSnapshot 抓取快照。看一下 GenerateSnapshot。

bool HeapSnapshotGenerator::GenerateSnapshot() {  Isolate* isolate = Isolate::FromHeap(heap_);  base::Optional<HandleScope> handle_scope(base::in_place, isolate);  v8_heap_explorer_.CollectGlobalObjectsTags();  // 抓取前先回收不用内存，保证看到的是存活的对象，否则影响内存泄漏的分析  heap_->CollectAllAvailableGarbage(GarbageCollectionReason::kHeapProfiler);  // 收集内存信息  snapshot_->AddSyntheticRootEntries();  FillReferences();  snapshot_->FillChildren();  

  return true;

}

GenerateSnapshot 的逻辑是首先进行GC 回收不用的内存，然后收集 GC 后的内存信息到 HeapSnapshot 对象。接着看收集完后的逻辑。

HeapSnapshotOutputStream stream(&m_frontend);  
snapshot->Serialize(&stream);

HeapSnapshotOutputStream 是用于通知调用方收集的数据（通过 m_frontend）。

explicit HeapSnapshotOutputStream(protocol::HeapProfiler::Frontend* frontend)      : m_frontend(frontend) {}  void EndOfStream() override {}  int GetChunkSize() override { return 102400; }  WriteResult WriteAsciiChunk(char* data, int size) override {    m_frontend->addHeapSnapshotChunk(String16(data, size));    m_frontend->flush();    return kContinue;  
}

HeapSnapshotOutputStream 通过 WriteAsciiChunk 告诉调用方收集的数据，但是目前我们还没有数据源，下面看看数据源怎么来的。

snapshot->Serialize(&stream);

看一下 Serialize。

void HeapSnapshot::Serialize(OutputStream* stream,                             HeapSnapshot::SerializationFormat format) const {  i::HeapSnapshotJSONSerializer serializer(ToInternal(this));  

  serializer.Serialize(stream);

}

最终调了 HeapSnapshotJSONSerializer 的 Serialize。

void HeapSnapshotJSONSerializer::Serialize(v8::OutputStream* stream) {  // 写者  writer_ = new OutputStreamWriter(stream);  // 开始写  

  SerializeImpl();

}

我们看一下  SerializeImpl。

void HeapSnapshotJSONSerializer::SerializeImpl() {  DCHECK_EQ(0, snapshot_->root()->index());  writer_->AddCharacter('{');  writer_->AddString("\"snapshot\":{");  SerializeSnapshot();  if (writer_->aborted()) return;  writer_->AddString("},\n");  writer_->AddString("\"nodes\":[");  SerializeNodes();  if (writer_->aborted()) return;  writer_->AddString("],\n");  writer_->AddString("\"edges\":[");  SerializeEdges();  if (writer_->aborted()) return;  writer_->AddString("],\n");  writer_->AddString("\"trace_function_infos\":[");  SerializeTraceNodeInfos();  if (writer_->aborted()) return;  writer_->AddString("],\n");  writer_->AddString("\"trace_tree\":[");  SerializeTraceTree();  if (writer_->aborted()) return;  writer_->AddString("],\n");  writer_->AddString("\"samples\":[");  SerializeSamples();  if (writer_->aborted()) return;  writer_->AddString("],\n");  writer_->AddString("\"locations\":[");  SerializeLocations();  if (writer_->aborted()) return;  writer_->AddString("],\n");  writer_->AddString("\"strings\":[");  SerializeStrings();  if (writer_->aborted()) return;  writer_->AddCharacter(']');  writer_->AddCharacter('}');  

  writer_->Finalize();

}

SerializeImpl 函数的逻辑就是把快照数据通过 OutputStreamWriter 对象 writer_ 写到 writer_ 持有的 stream 中。写的数据有很多种类型，这里以 AddCharacter 为例。

void AddCharacter(char c) {  chunk_[chunk_pos_++] = c;  

  MaybeWriteChunk();

}

每次写的时候都会判断是不达到阈值，是的话则先推给调用方。看一下 MaybeWriteChunk。

void MaybeWriteChunk() {  if (chunk_pos_ == chunk_size_) {    WriteChunk();  

  }

}

  

void WriteChunk() {

  // stream 控制是否还需要写入，通过 kAbort 和 kContinue  if (stream_->WriteAsciiChunk(chunk_.begin(), chunk_pos_) ==      v8::OutputStream::kAbort)    aborted_ = true;  

  chunk_pos_ = 0;

}

我们看到最终通过 stream 的 WriteAsciiChunk 写到 stream 中。

WriteResult WriteAsciiChunk(char* data, int size) override {  m_frontend->addHeapSnapshotChunk(String16(data, size));  m_frontend->flush();  

  return kContinue;

}

WriteAsciiChunk 调用 addHeapSnapshotChunk 通知调用方。

void Frontend::addHeapSnapshotChunk(const String& chunk){    v8_crdtp::ObjectSerializer serializer;    serializer.AddField(v8_crdtp::MakeSpan("chunk"), chunk);  

    frontend_channel_->SendProtocolNotification(v8_crdtp::CreateNotification("HeapProfiler.addHeapSnapshotChunk", serializer.Finish()));

}

触发 HeapProfiler.addHeapSnapshotChunk 事件，并传入快照的数据，最终触发 JS 层的事件。再看一下文章开头的代码。

let chunk = '';

const cb = (result) => {

  chunk += result.params.chunk;

};

  

session.on('HeapProfiler.addHeapSnapshotChunk', cb);  
session.post('HeapProfiler.takeHeapSnapshot', (err, r) => {  session.off('HeapProfiler.addHeapSnapshotChunk', cb);  

    console.log(err || chunk);

});

这个过程是否清晰了很多。从过程中也看到，抓取快照虽然传入了回调，但是其实是以同步的方式执行的，因为提交 HeapProfiler.takeHeapSnapshot 命令后，V8 就开始收集内存，然后不断触发 

HeapProfiler.addHeapSnapshotChunk 事件，直到堆数据写完，然后执行 JS 回调。

  

总结：整个过程不算复杂，因为我们没有涉及到堆内存管理那部分，V8 Inspector 提供了很多命令，有时间的话后续再分析其他的命令。

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/p1D0zKTkxp0sYzVhraibUb5ERqI9JzIVcic3M8aWubN46smHGOjicic3xYxUVLfdjKggqoX8hBk7tT2Wnw9KAxHGOQ/0?wx_fmt=jpeg)

theanarkh

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzUyNDE2OTAwNw==&mid=2247485886&idx=1&sn=62baf2c18d8fd0bf890b879ea0cde082&chksm=fa3033fecd47bae869117f3a6be4fbfc80ec2eb99dd86b135d38941f960bb6d6ae6d3eeaefec&mpshare=1&scene=24&srcid=1023T1YTU99k7b8IrKhe0ch3&sharer_sharetime=1634951283615&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0546cdd51c821d74b1718801c7d3854c7bea16cb17dde3e386aec1fd6602c9bf4000017d4407b5f7a659a00b8aff0de468f6dfec45473e6362600e17702eca11de83f2d3eddfefddeb111f3fcb32a01225562ee19273bb05dbe181a12ee3c34b9717e141a041ccf4a4e5a37f072ca19a5c946c62fcf4c47fe&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSERk9oqJ8EO%2Bj1kv1c0HiRLmAQIE97dBBAEAAAAAAMTnEIA4guYAAAAOpnltbLcz9gKNyK89dVj0fHauBMFzVPtX8Qu8T0pfyLBavhbH20%2FVNngadOfwtQNhrfWbuj%2BaSygLLmq%2FwA6LbPP8BvJKTHHMgpUA6f0KA3YBtO23H6Dm5Fw8R8wL6lhWIKBkWRpQpmbXDDVqskZSzQaBY7GFHPq%2FSnB4cfsCBQY2qEEGzQvP7WWjiiwuaH0cl0QE%2F51F%2F2hJpg981SFxAE5lwgwl6LBua7JBqvh7MuLF7hDT7updXskVBf2lcKdMVnCO8xCa3n3fjSqtU40a&acctmode=0&pass_ticket=d%2FfTAR4lVy9sitD6YhGtXeA8nxNf803H%2B6L88Mtrft%2Be4LJem5GM1dulnwTutEw8&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)钟意作者

Node.js154

V819

Node.js · 目录

上一篇深入理解 Node.js 的 Buffer下一篇Node.js 的底层原理

阅读 130

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/0vaZibicuhx1UxIriaZzSJL4lkkOb0XmGvbgZwoDAfkkvPyAJia0IDGjRN37a1pzJOfibmWicL928IaMhuDZlpqgVTrg/300?wx_fmt=png&wxfrom=18)

编程杂技

3分享在看

写留言

写留言

**留言**

暂无留言