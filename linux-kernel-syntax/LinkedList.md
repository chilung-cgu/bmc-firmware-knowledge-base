# Linux Kernel Linked List (`struct list_head`)

Linux 核心的鏈結串列 (Linked List) 實作是 C 語言程式設計中的一個經典案例。它與一般教科書上所教的實作方式**截然不同**。核心採用了一種「侵入式 (Intrusive)」的設計，允許將同一個資料結構同時串接在多個不同的鏈結串列中，且無需為每個串列重新定義節點結構。

---

## 1. 核心觀念差異：教科書 vs. Linux Kernel

### ❌ 傳統教科書做法 (內含指標)

傳統做法是將「指向下一個節點的指標」直接定義在資料結構中。

```c
struct fox {
    unsigned long tail_length;
    unsigned long weight;
    struct fox *next;  // <--- 指向下一個 struct fox
    struct fox *prev;  // <--- 指向上一個 struct fox
};
```

**缺點：**

1.  **重用性差**：如果你需要將 `struct fox` 放入另一個完全不同的串列（例如「受傷的狐狸清單」），你需要修改結構體定義，增加 `struct fox *next_injured`。
2.  **通用性差**：你無法編寫一個通用的 `sort_list()` 函數來排序任何類型的串列（因為每個結構體的 `next` 指標位置和型別都不同）。

### ✅ Linux Kernel 做法 (侵入式設計)

Linux Kernel 僅定義了一個通用的雙向鏈結結點：

```c
struct list_head {
    struct list_head *next, *prev;
};
```

然後，將這個 `list_head` **「嵌入」** 到你的資料結構中：

```c
struct fox {
    unsigned long tail_length;
    unsigned long weight;
    struct list_head list;         // <--- 串接在「所有狐狸」串列
    struct list_head injured_list; // <--- 串接在「受傷狐狸」串列
};
```

**關鍵差異圖解：**

- **傳統**：火車車廂頭尾相連。車廂本身就是鏈結的一部分。
- **Kernel**：像是用一條繩子串起一堆迴紋針。
  - `mctp_neigh` 就像是掛在迴紋針上的「名牌」。
  - `list_head` 就是那個「迴紋針」。
  - 所有的操作（新增、刪除、遍歷）都只針對「迴紋針 (`list_head`)」進行，完全不關心掛在下面的是什麼資料。

> **對於您的疑問**：
> "struct mctp_neigh 這種 node 的結構，不是結構內存在一個指標指向下一個 struct mctp_neigh 的開頭，而是有一個 struct list_head list，包含指向下一個 struct mctp_neigh 中 struct list_head list 中的指標？"
>
> **是的，完全正確！** `list.next` 指向的是下一個節點的 `list` 成員，而不是該節點的起始位址。

---

## 2. 魔法巨集：`container_of()`

既然 `list_head` 只指向下一個 `list_head`，那我們拿到一個 `list_head` 指標時，要怎麼找回它所屬的 `struct fox`（父結構）呢？

這就是 Linux Kernel 最著名的巨集 **`container_of`** 的功用。它的核心邏輯是：

> 「給我 **指標(ptr)**，我知道它是 **某種結構(type)** 裡面的 **某個成員(member)**，請幫我算出結構的起始位址！」

### 語法與參數拆解

```c
container_of(ptr, type, member)
```

**參數三要素：**

1.  **`ptr`** (指標)：
    - **意義**：你手上拿著的「迴紋針」位址。
    - **例子**：指向 `n->list` 的指標。
2.  **`type`** (結構型別)：
    - **意義**：你要找回的「大車廂」是什麼型別？
    - **例子**：`struct mctp_neigh`。
3.  **`member`** (成員名稱)：
    - **意義**：這個迴紋針在車廂裡面的「名字」叫什麼？
    - **例子**：`list`（因為我們在結構體裡定義 `struct list_head list;`）。

範例：

```c
// 假設我們有一個 struct mctp_neigh，其中包含了 list_head：
struct mctp_neigh {
    int eid;
    struct list_head list; // 這是我們嵌入的成員
};
struct list_head *ptr = ...; // 假設你從某個 API 拿到了這個 list 成員的位址

// 當你要找回大車廂時：
struct mctp_neigh *n = container_of(ptr, struct mctp_neigh, list);
```

### 實作原理 (進階)

了解用法後，我們來看看它的原始碼：

```c
#define container_of(ptr, type, member) ({          \
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

這段程式碼做了三件事：

1.  **型別檢查 (Type Checking)**：
    `const typeof( ((type *)0)->member ) *__mptr = (ptr);`
    - 利用 `typeof` 取得 `member` 的型別，並宣告一個暫時指標指向 `ptr`。如果傳入的指標型別不對，編譯器會報警告。

2.  **計算起始位址 (Pointer Arithmetic)**：
    `(char *)__mptr - offsetof(type,member)`
    - `offsetof(type, member)`：計算 `member` 在 `struct type` 裡面距離開頭有多少 bytes。
    - `(char *)__mptr`：轉成 `char *` 以進行 byte 級別的減法。
    - **相減**：`目前位址 - 偏移量 = 起始位址`。

3.  **轉型回父結構 (Cast)**：
    `(type *)( ... )`
    - 最後把算出來的位址，強制轉型回 `type *`。

### 圖解

```
Memory:
+----------------------+  <--- struct fox A 的起始位址 (我們想知道這個！)
| tail_length          |
| weight               |
+----------------------+
| struct list_head list|  <--- 我們手上有這個位址 (ptr)
|   next ---------------------> 指向 struct fox B 的 list
|   prev               |
+----------------------+
```

`container_of` 做的事情就是：拿 `list` 的位址，減去它前面成員 (`tail_length` + `weight`) 的大小，就回到了 `struct fox` 的門口。

---

## 3. 常見 API 與使用範例

### 定義與初始化

```c
// 1. 定義鏈結串列的「頭」(Head)
// 這通常放在 mctp_dev 結構裡
struct mctp_dev {
    // ...
    struct list_head neigh_list;
};

// 初始化 (必須做，否則 next/prev 是亂碼)
INIT_LIST_HEAD(&mdev->neigh_list);

// 2. 定義節點 (Node)
struct mctp_neigh {
    int eid;
    struct list_head list; // 嵌入 list_head
};
```

### 新增節點 (list_add / list_add_tail)

```c
struct mctp_neigh *new_node = kmalloc(sizeof(*new_node), GFP_KERNEL);
new_node->eid = 50;

// 將 new_node->list 加入到 mdev->neigh_list 之後
list_add(&new_node->list, &mdev->neigh_list);

// 或者加入到尾端
list_add_tail(&new_node->list, &mdev->neigh_list);
```

### 遍歷串列 (list_for_each_entry)

這是最常用的巨集，它自動幫你處理了 `container_of` 的轉換。

```c
struct mctp_neigh *cursor;

// cursor: 用來跑迴圈的指標 (會自動指向當前的 struct mctp_neigh)
// &mdev->neigh_list: 串列的頭
// list: 在 struct mctp_neigh 中叫什麼名字
list_for_each_entry(cursor, &mdev->neigh_list, list) {
    if (cursor->eid == 50) {
        printk("Found neighbor EID 50!\n");
        break;
    }
}
```

這個巨集展開後，其實就是一個 `for` 迴圈，裡面不斷呼叫 `container_of` 把 `list.next` 轉回 `struct mctp_neigh *`。

### 刪除節點 (list_del)

```c
list_del(&cursor->list);
kfree(cursor); // 記得釋放記憶體
```

---

## 4. 總結

Linux Kernel 的 Linked List 設計哲學：

1.  **通用性**：`struct list_head` 可以串接任何東西。
2.  **型別安全**：透過 `container_of` 和 `list_for_each_entry` 確保型別轉換正確。
3.  **雙向鏈結**：預設都是 Doubly Linked List，方便移除 (`prev->next = next`, `next->prev = prev`)，O(1) 時間複雜度。

這種設計雖然初學時覺得複雜（需要理解指標偏移量），但一旦習慣後，會發現它極度強大且靈活，是閱讀與撰寫 Kernel Code 的必備基礎知識。
