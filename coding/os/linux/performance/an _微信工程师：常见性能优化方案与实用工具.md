高可用架构
*2024 年 03 月 06 日 08:19* *广东*
以下文章来源于腾讯云开发者  ，作者张江涛

\](https://mp.weixin.qq.com/s?\_\_biz=MzAwMDU1MTE1OQ==&mid=2653562914&idx=1&sn=2d323c1eed2fd3e5870a7f15dea44237&chksm=8139b9bab64e30acbe7cee5c0ba0b5ce18c3cd1499dc53d5d92b2dcf9adb1dbe4659cbfee26b&mpshare=1&scene=24&srcid=0306PiwJ26g4tBzAalAFqZ3C&sharer_shareinfo=12b20ed0c86342c3e2541fe4823fbd9b&sharer_shareinfo_first=12b20ed0c86342c3e2541fe4823fbd9b&key=daf9bdc5abc4e8d0224d824ab0f15e3fe410b4d7ad060a071367539e786f45f213cc374130ca13ee3645bbc5e1469163e4d6e60971cec8ab9ae9eb653f8c0e1a6f3f9625f3c9d2a1e8a79815e0373154b4c4158af0f79fee5357f402f03ab9a812c45c54dc6769c604b0a1ea2612dabf310fc4edb4a28f8aacc9d6abcb0b2e26&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQcbYWwaW0aIYiLCIkcPbPExLmAQIE97dBBAEAAAAAAO%2FIK48bZFIAAAAOpnltbLcz9gKNyK89dVj0hUCzBaTe0RzHf3USdNpazxIS3qoTMtLAGuYjKtx%2BmiOim1C4m3MBXnc8er7D7dLCwVfm6S0rjaRvA48%2FlLTiUaqMB4to2mRtM2h8qw%2FKuPjRVIUlwal%2FDEh8e1Keb%2BcNO12shKQNqK6OD4mnclWBBzDwRJnb%2FAvKjRT4oqnYzcqGixKxaz3Ug6sTY20zzLU6qqBKURf%2B38SbA6LU%2FNNLcviGwr4FrQzJgLg5Ph1lP89EFX6EjAll3QWOYcL6fger&acctmode=0&pass_ticket=Q4VrPmg%2Fpqw9wgGdCLcn2FDa14aPpxoHJknq4dLU3QR2Md8oqXvll3GAiqHVa1vN&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

性能优化是降本增效路上必不可少的手段之一，在合适的时机采用合理的手段进行性能优化，一方面可以实现系统性能提升的目标，另一方面也可以借机对腐化的代码进行清理。在程序员的面试环节中，性能优化的问题也几乎是必考题。

然而性能优化并非一锤子买卖，需要一直监控，一直优化。过早的优化、过度的优化，以及优化  ROI  都是程序员们在工作中需要评估的关键点。本文作者总结了日常工作中常见的性能优化问题，围绕数据结构展开推荐了常见的几种性能优化方案——既有提升  3  倍性能的优化技巧，也有扛住 26  亿/s API  调用量的健壮方案。文末还推荐了三款好用的性能测试工具，值得点赞收藏！

# 01 使用 Class 代替 ProtoBuf 协议

在大量调用的 API 代码中尽量不要用  **ProtoBuf**  协议，最好使用 C++的  **Class**  来代替。因为  **ProtoBuf**  采用的是  **Arena 内存分配器**策略，有些场景会比 C++的 Class 内存管理复杂，当有大量内存分配和释放的时候会比 Class 的性能差很多。而且  **Protobuf**  会不断分配和回收**小内存对象**，持续地分配和删除小内存对象导致产生**内存碎片**，降低程序的内存使用率，尤其是当协议中包含 string 类型的时候，性能差距可能有几倍。对于包含了很多小对象的 Protobuf message，**析构过程**会导致堆上分配了许多对象，从而会变得复杂，导致析构函数执行速度变慢。

下面给出一个实际开发中使用 Class 代替 Protobuf 的例子：

**ProtoBuf**  协议：

```c
message Param {    optional string name = 1;    optional string value = 2;}message ParamHit {    enum Type {        Unknown = 0;        WhiteList = 1;        LaunchLayer = 2;        BaseAB = 3;        DefaultParam = 4;    }    optional Param param = 1;    optional uint64 group_id = 2;    optional uint64 expt_id = 3;    optional uint64 launch_layer_id = 4;    optional string hash_key_used = 5;    optional string hash_key_val_used = 6;    optional Type type = 7;    optional bool is_hit_mbox = 8;}
```

改写的  **Class ：**

```c
class ParamHitInfo {public:class Param {public:    Param() = default;    ~Param() = default;        const std::string & name() const {        return name_;    }    void set_name(const std::string &name) {        name_ = name;    }    void clear_name() {        name_.clear();    }    const std::string & value() const {        return value_;    }    void set_value(const std::string &value) {        value_ = value;    }    void clear_value() {        value_.clear();    }    void Clear() {        clear_name();        clear_value();    }private:    std::string name_, value_;};    ParamHitInfo() {        expt_id_ = group_id_ = launch_layer_id_ = 0u;        is_hit_mbox_ = false;        type_ = ParamHit::Unknown;    }    ~ParamHitInfo() = default;    void Clear() {        clear_group_id();        clear_expt_id();        clear_launch_layer_id();        clear_is_hit_mbox();        clear_hash_key_used();        clear_hash_key_val_used();        clear_type();        param_.Clear();    }    const ParamHit ToProtobuf() const {        ParamHit ans;        ans.set_expt_id(expt_id_);        ans.set_group_id(group_id_);        ans.set_launch_layer_id(launch_layer_id_);        ans.set_is_hit_mbox(is_hit_mbox_);        ans.set_hash_key_used(hash_key_used_);        ans.set_hash_key_val_used(hash_key_val_used_);        ans.set_type(type_);        ans.mutable_param()->set_name(param_.name());        ans.mutable_param()->set_value(param_.value());        return ans;    }    uint64_t group_id() const {        return group_id_;    }    void set_group_id(const uint64_t group_id) {        group_id_ = group_id;    }    void clear_group_id() {        group_id_ = 0u;    }    uint64_t expt_id() const {        return expt_id_;    }    void set_expt_id(const uint64_t expt_id) {        expt_id_ = expt_id;    }    void clear_expt_id() {        expt_id_ = 0u;    }    uint64_t launch_layer_id() const {        return launch_layer_id_;    }    void set_launch_layer_id(const uint64_t launch_layer_id) {        launch_layer_id_ = launch_layer_id;    }    void clear_launch_layer_id() {        launch_layer_id_ = 0u;    }    bool is_hit_mbox() const {        return is_hit_mbox_;    }    void set_is_hit_mbox(const bool is_hit_mbox) {        is_hit_mbox_ = is_hit_mbox;    }    void clear_is_hit_mbox() {        is_hit_mbox_ = false;    }    const std::string & hash_key_used() const {        return hash_key_used_;    }    void set_hash_key_used(const std::string &hash_key_used) {        hash_key_used_ = hash_key_used;    }    void clear_hash_key_used() {        hash_key_used_.clear();    }    const std::string & hash_key_val_used() const {        return hash_key_val_used_;    }    void set_hash_key_val_used(const std::string &hash_key_val_used) {        hash_key_val_used_ = hash_key_val_used;    }    void clear_hash_key_val_used() {        hash_key_val_used_.clear();    }    ParamHit_Type type() const {        return type_;    }    void set_type(const ParamHit_Type type) {        type_ = type;    }    void clear_type() {        type_ = ParamHit::Unknown;    }    const Param & param() const {        return param_;    }    Param * mutable_param() {        return &param_;    }    std::string ShortDebugString() const {        std::string ans = "type: " + std::to_string(type_);        ans.append(", group_id: ").append(std::to_string(group_id_));        ans.append(", expt_id: ").append(std::to_string(expt_id_));        ans.append(", launch_layer_id: ").append(std::to_string(launch_layer_id_));        ans.append(", hash_key_used: ").append(hash_key_used_);        ans.append(", hash_key_val_used: ").append(hash_key_val_used_);        ans.append(", param_name: ").append(param_.name());        ans.append(", param_val: ").append(param_.value());        ans.append(", is_hit_mbox: ").append(std::to_string(is_hit_mbox_));        return ans;    }    int ByteSize() {        int ans = 0;        ans += sizeof(uint64_t) * 3 + sizeof(bool) + sizeof(ParamHit_Type);        ans += hash_key_used_.size() + hash_key_val_used_.size() + param_.name().size() + param_.value().size();        return ans;    }private:    ParamHit_Type type_;    uint64_t group_id_, expt_id_, launch_layer_id_;    std::string hash_key_used_, hash_key_val_used_;    bool is_hit_mbox_;    Param param_;};
```

性能测试代码：

```c
TEST(ParamHitDestructorPerf, test) {    vector<ParamHit> hits;    vector<ParamHitInfo> hit_infos;    const int hit_cnts = 1000;    vector<pair<string, string>> params;    for (int i=0; i<hit_cnts; ++i) {        string name = "name: " + to_string(i);        string val;        int n = 200;        val.resize(n);        for (int i=0; i<n; ++i) val[i] = (i%10 + 'a');        params.push_back(make_pair(name, val));    }    int uin_start = 12345645;    for (int i=0; i<hit_cnts; ++i) {        ParamHit hit;        hit.set_expt_id(i + uin_start);        hit.set_group_id(i + 1 + uin_start);        hit.set_type(ParamHit::BaseAB);        hit.set_is_hit_mbox(false);        hit.set_hash_key_used("uin_bytes");        hit.set_hash_key_val_used(BusinessUtil::UInt64ToLittleEndianBytes(i));        auto p = hit.mutable_param();        p->set_name(params[i].first);        p->set_value(params[i].second);        hits.emplace_back(std::move(hit));    }    for (int i=0; i<hit_cnts; ++i) {        ParamHitInfo hit;        hit.set_expt_id(i + uin_start);        hit.set_group_id(i + 1 + uin_start);        hit.set_type(ParamHit::BaseAB);        hit.set_is_hit_mbox(false);        hit.set_hash_key_used("uin_bytes");        hit.set_hash_key_val_used(BusinessUtil::UInt64ToLittleEndianBytes(i));        auto p = hit.mutable_param();        p->set_name(params[i].first);        p->set_value(params[i].second);        hit_infos.emplace_back(std::move(hit));    }    int kRuns = 1000;    chrono::high_resolution_clock::time_point t1 = chrono::high_resolution_clock::now();    {        for (int i=0; i<kRuns; ++i) {            for (auto &&hit: hits) {                auto tmp = hit;            }        }    }    chrono::high_resolution_clock::time_point t2 = chrono::high_resolution_clock::now();    auto time_span = chrono::duration_cast<chrono::milliseconds>(t2 - t1);    std::cerr << "ParamHit_PB Destructor kRuns: " << kRuns << " hit_cnts: " << hit_cnts << " cost: " << time_span.count() << "ms\n";        t1 = chrono::high_resolution_clock::now();    {        for (int i=0; i<kRuns; ++i) {            for (auto &&hit: hit_infos) {                auto tmp = hit;            }        }    }    t2 = chrono::high_resolution_clock::now();    time_span = chrono::duration_cast<chrono::milliseconds>(t2 - t1);    std::cerr << "ParamHitInfo_Class Destructor kRuns: " << kRuns << " hit_cnts: " << hit_cnts << " cost: " << time_span.count() << "ms\n";}
```

性能对比结果：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到使用 C++的 Class 相比于 ProtoBuf 可以提升 3 倍的性能。

# 02

使用 Cache Friendly 的数据结构

这里想先抛出一个问题：使用哈希表的查找一定比使用数组的查找快吗？

Q：理论上来说哈希表的查找复杂度是 O(1)，数组的查找复杂度是 O(n)，那么是不是可以得到一个结论就是说哈希表的查找速度一定比数组快呢？

A：其实是不一定的，由于数组具有较高的缓存局部性，可提高 CPU 缓存的命中率，所以在有些场景下数组的查找效率要远远高于哈希表。

这里给出一个常见操作耗时的数据（2020 年）：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

下面也给出一个项目中的使用 Cache Friendly 优化的例子：

优化前的数据结构：

```c
class HitContext {public:    inline void update_hash_key(const std::string &key, const std::string &val) {        hash_keys_[key] = val;    }    inline const std::string * search_hash_key(const std::string &key) const {        auto it = hash_keys_.find(key);        return it != hash_keys_.end() ? &(it->second) : nullptr;    }private:    Context context_;    std::unordered_map<std::string, std::string> hash_keys_;};
```

优化后的数据结构：

```c
class HitContext {public:    inline void update_hash_key(const std::string &key, const std::string &val) {        if (Misc::IsSnsHashKey(key)) {            auto sns_id = Misc::FastAtoi(key.c_str()+Misc::SnsHashKeyPrefix().size());            sns_hash_keys_.emplace_back(sns_id, Misc::LittleEndianBytesToUInt32(val));            return;        }        hash_keys_[key] = val;    }    inline void update_hash_key(const std::string &key, const uint32_t val) {        if (Misc::IsSnsHashKey(key)) {            auto sns_id = Misc::FastAtoi(key.c_str()+Misc::SnsHashKeyPrefix().size());            sns_hash_keys_.emplace_back(sns_id, val);            return;        }        hash_keys_[key] = Misc::UInt32ToLittleEndianBytes(val);    }    inline const std::string search_hash_key(const std::string &key, bool &find) const {        if (Misc::IsSnsHashKey(key)) {            auto sns_id = Misc::FastAtoi(key.c_str()+Misc::SnsHashKeyPrefix().size());            auto it = std::find_if(sns_hash_keys_.rbegin(), sns_hash_keys_.rend(), [sns_id](const std::pair<uint32_t, uint32_t> &v) { return v.first == sns_id; });            find = it != sns_hash_keys_.rend();            return find ? Misc::UInt32ToLittleEndianBytes(it->second) : "";        }        auto it = hash_keys_.find(key);        find = it != hash_keys_.end();        return find ? it->second : "";    }private:    Context context_;    std::unordered_map<std::string, std::string> hash_keys_;    std::vector<std::pair<uint32_t, uint32_t>> sns_hash_keys_;};
```

性能测试代码：

```c
TEST(HitContext, test) {    const int keycnt = 264;    std::vector<std::string> keys, vals;    for (int j = 0; j < keycnt; ++j) {        auto key = j+21324;        auto val = j+94512454;        keys.push_back("sns"+std::to_string(key));        vals.push_back(std::to_string(val));    }    const int kRuns = 1000;    std::unordered_map<uint32_t, uint64_t> hash_keys;    chrono::high_resolution_clock::time_point t1 = chrono::high_resolution_clock::now();    for (int i = 0; i < kRuns; ++i) {        HitContext1 ctx;        for (int j = 0; j < keycnt; ++j) {           ctx.update_hash_key(keys[j], vals[j]);        }        for (int j=0; j<keycnt; ++j) {            auto val = ctx.search_hash_key(keys[j]);            if (!val) assert(0);        }    }    chrono::high_resolution_clock::time_point t2 = chrono::high_resolution_clock::now();    auto time_span = chrono::duration_cast<chrono::microseconds>(t2 - t1);    std::cerr << "HashTable Hitcontext cost: " << time_span.count() << "us" << std::endl;    hash_keys.clear();    t1 = chrono::high_resolution_clock::now();    for (int i = 0; i < kRuns; ++i) {        HitContext2 ctx;        for (int j = 0; j < keycnt; ++j) {           ctx.update_hash_key(keys[j], vals[j]);        }        for (int j=0; j<keycnt; ++j) {            bool find = false;            auto val = ctx.search_hash_key(keys[j], find);            if (!find) assert(0);        }    }    t2 = chrono::high_resolution_clock::now();    time_span = chrono::duration_cast<chrono::microseconds>(t2 - t1);    std::cerr << "Vector HitContext cost: " << time_span.count() << "us" << std::endl;}
```

性能对比结果：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 03

使用 jemalloc/tcmalloc 代替普通的 malloc 方式

由于代码中大量使用了 C++的 STL，所以会出现以下几种缺点：

|                                                                                                                                                                                                                                                                                               |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| \*\*内存碎片：\*\*频繁分配和释放不同大小的对象，可能导致内存碎片，降低内存的使用效率。<br><br>\*\*Cache 不友好：\*\*而且 STL 的普通内存分配器分散了对象的内存地址，降低了数据的缓存命中率。<br><br>\*\*并发差：\*\*STL 的默认内存分配器可能使用全局锁，相当于给加了一把大锁，在多线程环境下性能表现很差。 |

目前在我们的代码中加 jemalloc 还是很方便的，就是在所编译的 target 中加下依赖就好了，比如：

```c
cc_library(name = "mmexpt_dye_api",srcs = ["mmexpt_dye_api.cc",],hdrs = ["mmexpt_dye_api.h",],includes = ['.'],deps = ["//mm3rd/jemalloc:jemalloc",],copts = ["-O3","-std=c++11",],linkopts = [],visibility = ["//visibility:public"],)
```

使用 jemalloc 与不使用 jemalloc 前后性能对比（这里的测试场景是在 loadbusiness 的时候，具体涉及到了一些业务代码）

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以发现使用 jemalloc 可以提升 20%多的性能，还是优化了很大的，很小的开发成本（只需要加一个编译依赖）带来不错的收益。

# 04

使用无锁数据结构

在过去项目开发的时候使用过一种双 buffer 的无锁数据结构，之所以使用双 buffer 是因为 API 有大约 26 亿/s 的调用量，这么高的调用量对性能的要求是很高的。数据结构的定义：

```c
struct expt_api_new_shm {  void *p_shm_data;  // header  volatile int *p_mem_switch; // 0:uninit. 1:mem 1 on server. 2:mem 2 on server  uint32_t *p_crc_sum;  // data  expt_new_context* p_new_context;  parameter2business* p_param2business;  char* p_business_cache;  HashTableWithCache hash_table; //多级哈希表};
```

常用的几个函数：

```c
int InitExptNewShmData(expt_api_new_shm *pstShmData, void *pData) {  int ptr_offset = EXPT_NEW_SHM_HEADER_SIZE;  pstShmData->p_shm_data = pData;  pstShmData->p_mem_switch = MAKE_PTR(volatile int *, pData, 0);  pstShmData->p_crc_sum = MAKE_PTR(uint32_t *, pData, 4);  pstShmData->p_new_context =      (expt_new_context *)((uint8_t *)pstShmData->p_shm_data + ptr_offset);  pstShmData->p_param2business =      (parameter2business *)((uint8_t *)pstShmData->p_shm_data + ptr_offset +                             EXPT_NEW_SHM_OFFSET_0);  pstShmData->p_business_cache =      (char *)((uint8_t *)pstShmData->p_shm_data + ptr_offset +                            EXPT_NEW_SHM_OFFSET_1);    size_t node_size = sizeof(APICacheItem), row_cnt = sizeof(auModsInCache)/sizeof(size_t);  size_t hash_tbl_size = CalHashTableWithCacheSize(node_size, row_cnt, auModsInCache);  pstShmData->hash_table.pTable = (void *)((uint8_t *)pstShmData->p_shm_data + EXPT_NEW_SHM_SIZE - hash_tbl_size);  int ret = HashTableWithCacheInit(&pstShmData->hash_table, hash_tbl_size, node_size, row_cnt, auModsInCache);  return ret;}int ResetExptNewShmData(expt_api_new_shm *pstShmData) {  int iOffset = 0;  if (*pstShmData->p_mem_switch <= 1) {    iOffset = 0;  } else if (*pstShmData->p_mem_switch > 1) {    iOffset = EXPT_NEW_SHM_DATA_SIZE;  }  void *ptrData = MAKE_PTR(void *, pstShmData->p_shm_data,                           EXPT_NEW_SHM_HEADER_SIZE + iOffset);  memset(ptrData, 0, EXPT_NEW_SHM_DATA_SIZE);  return 0;}int ResetExptNewShmHeader(expt_api_new_shm *pstShmData) {  memset(pstShmData->p_shm_data, 0, EXPT_NEW_SHM_HEADER_SIZE);  return 0;}void SwitchNewShmMemToWrite(expt_api_new_shm *pstShmData) {  int iSwitchOffset =      EXPT_NEW_SHM_DATA_SIZE * ((*pstShmData->p_mem_switch <= 1 ? 0 : 1));  int ptr_offset = EXPT_NEW_SHM_HEADER_SIZE + iSwitchOffset;  pstShmData->p_new_context =      (expt_new_context *)((uint8_t *)pstShmData->p_shm_data + ptr_offset);  pstShmData->p_param2business =      (parameter2business *)((uint8_t *)pstShmData->p_shm_data + ptr_offset +                             EXPT_NEW_SHM_OFFSET_0);  pstShmData->p_business_cache =      (char *)((uint8_t *)pstShmData->p_shm_data + ptr_offset +                            EXPT_NEW_SHM_OFFSET_1);}void SwitchNewShmMemToWriteDone(expt_api_new_shm *pstShmData) {  if (*pstShmData->p_mem_switch <= 1)    *pstShmData->p_mem_switch = 2;  else    *pstShmData->p_mem_switch = 1;}void SwitchNewShmMemToRead(expt_api_new_shm *pstShmData) {  int iSwitchOffset =      EXPT_NEW_SHM_DATA_SIZE * ((*pstShmData->p_mem_switch <= 1 ? 1 : 0));  int ptr_offset = EXPT_NEW_SHM_HEADER_SIZE + iSwitchOffset;  pstShmData->p_new_context =      (expt_new_context *)((uint8_t *)pstShmData->p_shm_data + ptr_offset);  pstShmData->p_param2business =      (parameter2business *)((uint8_t *)pstShmData->p_shm_data + ptr_offset +                             EXPT_NEW_SHM_OFFSET_0);  pstShmData->p_business_cache =      (char *)((uint8_t *)pstShmData->p_shm_data + ptr_offset +                            EXPT_NEW_SHM_OFFSET_1);}
```

双 buffer 的工作原理就是：设置两个 buffer，一个用于读，另一个用于写。

|                                                                                                                                                                                                                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| - 初始化这两个 buffer 为空，调用 InitExptNewShmData 函数。<br> <br>- 对于写操作，先准备数据，即调用 SwitchNewShmMemToWrite 函数，等数据准备完(即写完相应的数据)，然后调用 SwitchNewShmMemToWriteDone 函数，完成指针的切换。<br> <br>- 对于读操作，线程从读 buffer 读取数据，调用 SwitchNewShmMemToRead 函数。 |

我们平台的场景主要是读，而且由于拉取实验配置采用的都是增量的拉取方式，所以配置的改变也不是很频繁，也就很少有写操作的出现。采用双 buffer 无锁数据结构的优势在于可以提高并发性能，由于读写操作在不同的 buffer 上同时进行，所以不需要额外加锁，减少了数据竞争和锁冲突的可能性。当然这种数据结构也有相应的缺点，就是会多用了一倍的内存，用空间换时间。

# 05

对于特定的场景采用特定的处理方式

这其实也很容易理解，有很多场景是需要定制化优化的，所以不能从主体代码的层面去优化了，那换个思路，是不是可以从返回的数据格式进行优化呢？举个我们过去遇到的一个例子：我们平台有一个染色场景，就是需要对当天登录的所有微信用户计算命中情况，旧的数据格式其实返回了一堆本身染色场景不需要的字段，所以这里其实是可以优化的。

优化前的数据格式：

```c
struct expt_param_item {    int experiment_id;    int expt_group_id;    int layer_id;    int domain_id;    uint32_t seq;    uint32_t start_time;    uint32_t end_time;    uint8_t  expt_type;    uint16_t  expt_client_expand;    int parameter_id;    uint8_t value[MAX_PARAMETER_VLEN];    char param_name[MAX_PARAMETER_NLEN];    int value_len;    uint8_t is_pkg = 0;    uint8_t is_white_list = 0;    uint8_t is_launch = 0;    uint64_t bucket_src = 0;    uint8_t is_control = 0;};
```

其实染色场景下不需要参数的信息，只保留实验 ID、组 ID 以及分桶的信息就好了。优化后的数据格式：

```c
struct DyeHitInfo {    int expt_id, group_id;    uint64_t bucket_src;    DyeHitInfo(){}  DyeHitInfo(int expt_id_, int group_id_, uint64_t bucket_src_) :expt_id(expt_id_), group_id(group_id_), bucket_src(bucket_src_){}    bool operator <(const DyeHitInfo &hit) const {        if (expt_id == hit.expt_id) {            if (group_id == hit.group_id) return bucket_src < hit.bucket_src;            return group_id < hit.group_id;        }    return expt_id < hit.expt_id;  }    bool operator==(const DyeHitInfo &hit) {        return hit.expt_id == expt_id && hit.group_id == group_id && hit.bucket_src == bucket_src;    }    std::string ToString() const {        char buf[1024];        sprintf(buf, "expt_id: %u, group_id: %u, bucket_src: %lu", expt_id, group_id, bucket_src);        return std::string(buf);    }};
```

优化前后性能对比：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以其实针对某些特殊场景做一些定制化的开发成本也没有很高，但是带来的收益却是巨大的。

# 06

善用性能测试工具

这里列举一些常见的性能测试工具：linux 提供的 perf、GNU 编译器提供的 gprof、Valgrind、strace 等等。

这里推荐几个觉得好用的工具：

perf（linux 自带性能测试工具）

https://godbolt.org/（可以查看代码对应的汇编代码）

微信运维提供的性能测试工具：

https://github.com/brendangregg/FlameGraph （生成火焰图的工具）

# 07

总结

其实还有一些性能优化的地方，比如使用合适的数据结构和算法，减少大对象的拷贝，减少无效的计算，IO 与计算分离，分支预测等等，后续如果有时间的话可以再更新一篇新的文章。性能优化不是一锤子买卖，所以需要一直监控，一直优化。需要注意的一点是不要过度优化，在提升程序性能的时候不要丢掉代码的可维护性，而且还要评估下性能提升带来的收益是否与花费的时间成正比。总之，性能优化，长路漫漫。如果觉得本篇文章的内容对你有帮助，欢迎转发分享。

-End-

原创作者｜张江涛

**参考阅读：**

- [Spring 七种事务传播性介绍](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653562834&idx=1&sn=496e41b1ba83de558971bf1505d5ed0e&chksm=8139b64ab64e3f5ce8f4c138213453b5b538fba79ea8396866c57e24952fa69b63a375198903&scene=21#wechat_redirect)

- [Sora：技术细节推测与原理解读，行业影响与成功关键](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653562841&idx=1&sn=6c0a359b29872272cc6cea41884e6764&chksm=8139b641b64e3f573c1da8a8e05d94454630b922e8f98a86da4ad071e931327b528a2b9e8486&scene=21#wechat_redirect)

- [万字长文：一文详解单元测试干了什么](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653562876&idx=1&sn=00b9fbfb2132b79790ac822717634295&chksm=8139b664b64e3f7244a925a049989fdfcb2edb4d349b9d567ae6db00473872d165c9aeff9be9&scene=21#wechat_redirect)

- [vivo 在离线混部探索与实践](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653562905&idx=1&sn=505028bbb802e6d7c5432c0dbd3c7f50&chksm=8139b981b64e3097f9a8a65ce9a6d4b5abadb29ae28271bf8d54c19f23a6ec96db2df1b588fe&scene=21#wechat_redirect)

- [千万级高性能长连接 Go 服务架构实践](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653562694&idx=1&sn=9b111722c3d3edb14f32f7178a22590e&chksm=8139b6deb64e3fc8e0cdc6584c31caddf67a4689910088b8e8d7d3e94fcf28a66db46761db7a&scene=21#wechat_redirect)

本文由高可用架构转载。技术原创及架构实践文章，欢迎通过公众号菜单「联系我们」进行投稿

![](http://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapONl06YmHad4csRU93kcbJ76JIWzEAmOSVooibFHHkzfWzzkc7dpU4H06Wp9F6Z687vIghdawxvl47A/300?wx_fmt=png&wxfrom=19)

**高可用架构**

高可用架构公众号。

439 篇原创内容

公众号

阅读  4324

​

写留言

**留言 1**

- ANAs Not Anya
  日本 3 月 7 日
  赞
  第一个用类替代 protobuf，会不会有点本末倒置了。protobuf 一般用在微服务架构，用类替代了性能确实会提高，但通信成本和复杂度也会增加吧

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapONl06YmHad4csRU93kcbJ76JIWzEAmOSVooibFHHkzfWzzkc7dpU4H06Wp9F6Z687vIghdawxvl47A/300?wx_fmt=png&wxfrom=18)

高可用架构

71831

1

写留言

**留言 1**

- ANAs Not Anya
  日本 3 月 7 日
  赞
  第一个用类替代 protobuf，会不会有点本末倒置了。protobuf 一般用在微服务架构，用类替代了性能确实会提高，但通信成本和复杂度也会增加吧

已无更多数据
