
Linux爱好者

 _2024年02月23日 11:50_ _浙江_

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=19)

**Linux爱好者**

点击获取《每天一个Linux命令》系列和精选Linux技术资源。「Linux爱好者」日常分享 Linux/Unix 相关内容，包括：工具资源、使用技巧、课程书籍等。

75篇原创内容

公众号

> 转自：网络

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/icRxcMBeJfcibMzmLqtuTibQK1Taiciahb6qLa976iaSBZicTJtdiboMRmUXsZZUuoOVPiacTlYRaUUsVLGtNPjicdGA6V5w/640?wx_fmt=jpeg&from=appmsg&wxfrom=13)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/icRxcMBeJfcibMzmLqtuTibQK1Taiciahb6qLUYAr7s55aTHctibkslJAia4tQSUx6JOGFSUzKsD2LZ18fB2rmTVDG0rw/640?wx_fmt=jpeg&from=appmsg&wxfrom=13)

hashmap 之链地址法

# 1、定义哈希表 及 哈希桶 结构体

```
#include <stdio.h>#include <stdlib.h>#include <string.h>// 定义哈希桶的节点结构体typedef struct Node {    char* key;    int value;    struct Node* next;} Node;// 定义哈希表结构体typedef struct HashMap {    int size;    Node** buckets;} HashMap;
```

# 2、创建指定大小的哈希表

```
// 创建指定大小的哈希表HashMap* createHashMap(int size) {    HashMap* map = (HashMap*)malloc(sizeof(HashMap));    map->size = size;    map->buckets = (Node**)calloc(size, sizeof(Node*));    return map;}
```

# 3、哈希函数

```
// 哈希函数int hash(HashMap* map, char* key) {    int sum = 0;    for (int i = 0; i < strlen(key); i++) {        sum += key[i];    }    return sum % map->size;}
```

# 4、HashMap put操作

```
void put(HashMap* map, char* key, int value) {    Node* newNode = (Node*)malloc(sizeof(Node));    newNode->key = strdup(key);    newNode->value = value;    newNode->next = NULL;    int index = hash(map, key);    Node* curr = map->buckets[index];    if (curr == NULL) {        map->buckets[index] = newNode;    } else {        while (curr->next != NULL) {            curr = curr->next;        }        curr->next = newNode;    }}
```

# 5、HashMap get操作

```
// 从哈希表中获取指定键的值int get(HashMap* map, char* key) {    int index = hash(map, key);    Node* curr = map->buckets[index];    while (curr != NULL) {        if (strcmp(curr->key, key) == 0) {            return curr->value;        }        curr = curr->next;    }    return -1;  // 如果没有找到，返回 -1}
```

# 6、释放内存

```
// 释放哈希表的内存void freeHashMap(HashMap* map) {    for (int i = 0; i < map->size; i++) {        Node* curr = map->buckets[i];        while (curr != NULL) {            Node* temp = curr;            curr = curr->next;            free(temp->key);            free(temp);        }    }    free(map->buckets);    free(map);}
```

# 7、main方法测试

```
int main() {    HashMap* map = createHashMap(10);char a[] = "apple",b[] = "banana",o[] = "orange",w[] = "watermelon";    put(map, a, 1);    put(map, b, 2);    put(map, o, 3);    printf("Value of 'apple': %d\n", get(map, a));    printf("Value of 'banana': %d\n", get(map, b));    printf("Value of 'orange': %d\n", get(map, o));    printf("Value of 'watermelon': %d\n", get(map, w));    freeHashMap(map);    return 0;}
```

运行结果：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

result

该HashMap 使用了链地址法来处理冲突，即在哈希桶中的每个位置存储一个链表，哈希冲突时将键值对添加到链表的末尾。

createHashMap 函数创建了一个指定大小的哈希表，put 函数向哈希表中插入键值对，get 函数从哈希表中获取指定键的值，freeHashMap 函数用于释放哈希表的内存。在 main 函数中我们可以看到如何使用这个 HashMap 来存储和获取键值对的方式。

推荐阅读  点击标题可跳转

1、[内存统计与监控，你知多少？](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666569880&idx=2&sn=7644a2d092b4f71fc6127a31afac2c31&chksm=80dc6433b7abed256bc578a7a96ec67cd220e9c194b93e800621d9d8b267b3c1fad8e25277f3&scene=21#wechat_redirect)

2、[Linux 问题故障定位的技巧大全](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666569846&idx=2&sn=0dd147a011e0f3ecdbbc9389a56d5e52&chksm=80dc64ddb7abedcba32ccd860875d684016b3b11462548dffad9f61128c18686b209e3a951a4&scene=21#wechat_redirect)

3、[温故知新 | C 语言最全入门笔记！](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666569784&idx=2&sn=51ad870d75c87a6e1e0d1c057ed87090&chksm=80dc6493b7abed8552f33b448e1e33f53d163cb3eb7c40fd2275112b89b2d790ca42ce559915&scene=21#wechat_redirect)

阅读 1416

​

喜欢此内容的人还喜欢

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=18)

Linux爱好者

1352在看

写留言

写留言

**留言**

暂无留言