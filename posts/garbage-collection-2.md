<!--
.. title: Garbage Collection (2)
.. slug: garbage-collection-2
.. date: 2012/08/03 04:08:22
.. tags:
.. link:
.. description:
.. type: text
-->

上一篇提到了 reference counting 以及 weak reference. 這篇我將以 C++11 的 smart pointer 為骨幹做介紹, 但核心概念都是差不多的.

C++11 引入了一系列的 smart pointer, 使用 RAII (Resource Acquisition Is Initialization) 手法來確保資源在生命週期結束後的回收行為.

想法非常簡單: 配置在 stack 上的變數, 在離開 scope 後一定會呼叫 destructor, 因此只要設計一系列的容器, 讓它們在 destructor 裡做回收行為, 就可以讓編譯器為我們回收資源. 而 C++11 的 smart pointer 並不只是自動化的 new/delete 容器, 由於其 allocator/deleter 可自訂, programmer 實際上可用它來管理任何種類的資源.

## Strong Reference

在 C++ 裡: `std::shared_ptr` 即是使用 strong reference, 也是最常被用到的 smart pointer. 用法如下:

```cpp
std::shared_ptr<Object> sp(new Object);
sp.get();                        // get raw pointer
sp.reset();                      // ref count -1, set to null pointer
sp.reset(new Object);            // manage another pointer
std::shared_ptr<Object> sp2(sp); // ref count +1
```

用同一個 pointer 建立兩個不同的 `std::shared_ptr` 是錯誤用法, 會造成一次以上的回收:

```cpp
Object o = new Object;
std::shared_ptr<Object> sp1(o);
std::shared_ptr<Object> sp2(o); // bad
```

## Weak Reference

在 C++11, weak reference 對應的 smart pointer 是 `std::weak_ptr`. 前篇提到了 weak reference 可以用來解決 cyclic reference, 那麼是怎麼做到的呢? 很簡單, 因為 weak reference 不參與 reference counting, 也就是說它的生成, 複製, 銷毀都不會影響計數器, 自然也不會做任何回收的動作.
那麼為什麼我們需要 weak reference, 而不是簡單地用 raw pointer 呢? 差別就在 weak reference 雖然不參與 reference counting, 但它仍然知道該 pointer 目前的計數器資訊, 這讓它可以確認資源是否已被回收, 並在需要時暫時轉換為 strong reference. 這在多緒環境時尤其重要:

```cpp
void thread1 () {
    // ...
    // sp is a std::shared_ptr
    sp.reset(); // (1)
    // ...
}

void thread2 () {
    // ...
    // wp is a std::weak_ptr
    wp.get() // (2)
    // ...
}
```

如果 `std::weak_ptr` 跟 `std::shared_ptr` 一樣只用 `get()` 取得 raw pointer, 那麼在上述的例子中, (1) 如果觸發了回收, 那麼 (2) 拿到的 pointer 就會因而腐敗. 因此 `std::weak_ptr` 必須要使用 `lock()` 函式取得一個 `std::shared_ptr`, 讓計數器暫時不歸零, 就可以避免這個狀況:

```cpp
{
    std::shared_ptr<Object> sp = wp.lock();
    // counter will not be 0 in this scope
}
```

## Strong Reference From This

考慮以下的二元樹節點實作:

```cpp
class Node {
public:
    void setParent (std::weak_ptr<Node> n) {
        this->p = n;
    }
    void setLeft (std::shared_ptr<Node> n) {
        this->l = n;
        n->setParent(std::shared_ptr<Node>(this));
    }
    void setRight (std::shared_ptr<Node> n) {
        this->r = n;
        n->setParent(std::shared_ptr<Node>(this));
    }
private:
    std::weak_ptr<Node> p;
    std::shared_ptr<Node> l, r;
};
```

以上的程式片段犯了大忌: 直接使用 this 建立 `std::shared_ptr`, 導致離開該函式後會讓計數器馬上歸零, this 被意外地 delete.
由於我們的確需要取得目前正在管理 this 的 `std::shared_ptr`, 這時我們需要使用 `std::enable_shared_from_this`. 繼承自 `std::enable_shared_from_this` 的類別, 可以使用 `shared_from_this()` 函式取得管理 this 的 `std::shared_ptr`. 實作原理基本上是儲存一個 `std::weak_ptr`, 外部建立 `std::shared_ptr` 時更新這個 `std::weak_ptr`, 呼叫 `shared_from_this()` 時再轉換回 `std::shared_ptr`. 用法如下:

```cpp
class Node: public std::enable_shared_from_this<Node> {
public:
    void setParent (std::weak_ptr<Node> n) {
        this->p = n;
    }
    void setLeft (std::shared_ptr<Node> n) {
        this->l = n;
        n->setParent(this->shared_from_this());
    }
    void setRight (std::shared_ptr<Node> n) {
        this->r = n;
        n->setParent(this->shared_from_this());
    }
private:
    std::weak_ptr<Node> p;
    std::shared_ptr<Node> l, r;
};
```

## Unique Reference

在特殊情況下, 可能會需要確保同一時間只有一個物件有擁有權, 這時候 `std::unique_ptr` 就派上用場了. `std::unique_ptr` 是 `std::auto_ptr` 的改良版本. `std::auto_ptr` 的複製行為是「摧毀式複製」, 發生複製時會轉移資源的擁有權, 這使得它無法被用於標準容器, 並且在作為參數傳入函式時, 會意外地被轉移擁有權, 並在函式結束時觸發回收:

```cpp
void f1 () {
    std::auto_ptr<Object> a(new Object);
    // ownership moved into f2
    f2(a);
}

void f2(std::auto_ptr<Object> b) {
    // destructs b and delete pointee
}
```

`std::unique_ptr` 改良了這點, 它不具有 copy 語意, 但具有 move 語意. 需要複製時必須要明確使用 `std::move` 轉移擁有權.
