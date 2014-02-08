<!--
.. title: Garbage Collection (3)
.. slug: garbage-collection-3
.. date: 2014/02/08 08:40:30
.. tags:
.. link:
.. description:
.. type: text
-->

上一次談到了 C++11 的 smart pointer, 這次來談談 Qt. 在 Qt 你可以使用的工具有:

* Smart Pointers
    * `QPointer`
    * `QSharedPointer`
    * `QWeakPointer`
    * `QScopedPointer`
    * `QScopedArrayPointer`

* Based on QObject
    * parent-child relation
    * `QObject::deleteLater()`

* Copy On Write
    * `QSharedData`
    * `QSharedDataPointer`
    * `QExplicitlySharedDataPointer`

## Smart Pointers

用途基本上跟 STL, Boost 版本差不多, 但細部設計方針有些許不同; 例如 Qt 的版本都提供 raw pointer 的隱式轉換.

不同的 library 在這方面經常會有點出入, 像 Loki 的 smart pointer 就不使用成員函式, 一律使用全域函式對 smart pointer 本身操作.

由於用法都差不多, 本篇也不贄述, 請自行參閱相關文件.

## QObject

如果類別是一個 `QObject`, 就適用這些方式.

### Parent-Child relation

`QObject` 之間有 parent-child 關係, 你可以在 constructor 指定 parent, 該物件就會自動成為 child. Parent 在回收時會負責回收所有的 children, 因此 `QObject` 物件之間會是樹狀圖, 只要記得回收根部就可以回收整個體系.

這種回收方式又特別適合 `QWidget` 這種 GUI 類別, 因為它們很少出現環狀參用, 且有很明確的 composition 關係. 仔細回想一下 Qt 的範例是否都是在 main 建立一個 GUI 實體, 該實體所擁有的各種元件都只使用 new 無 delete? 就是因為有這種回收機制存在.

但這 parent-child 關係其實存在某種限制. 起因在 `QObject` 擁有與 event loop 溝通的能力, 其中一個特性就是它有 thread affinity. 講白話一點就是每個 `QObject` 都屬於某個 thread, 這樣在 `QObject` 發 signals 的時候, event loop 才知道這事件是來自哪一個 thread, 有沒有必要丟進 queue 裡分派等等.

`QObject` 屬於哪個 thread, 是看建立該物件時處在哪個 thread 上. 建立完成的實體也可以用 `QObject::moveToThread()` 丟到其他 thread 上; 注意, **只能「推」, 不能「拉」**, 因為你並不知道別的 thread 目前執行到哪裡, 自然也不能任意修改物件.

噢, 講得有點遠了. 這裡的真正問題是: 如果 parent 的 thread 和 child 不一樣會發生什麼事?

唔, 會發生很多事, 但絕對都不是好事. 一個很明顯的問題是回收機制不再安全. 你永遠不應該直接操作不同 thread 上的 `QObject`. 如果你使用的是 QRunnable 就更要小心: `QRunnable::run()` 是跑在不同的 thread 上!

### QObject::deleteLater()

當你呼叫了 `QObject::deleteLater()` 後, 該 QObject 並不會立刻回收, 而是通知 event queue 請它之後幫你回收. 它具有幾個特性:

* 多次呼叫不會重覆回收
* 保證 thread-safe, 它會在自己的 thread 回收
* 它是 slot

最後一點即是它和 smart pointer 的不同處: smart pointer 是依賴 variable scope 觸發, 而 `QObject::deleteLater()` 則可以交由特定事件觸發回收.

基本上只要需要手動回收通常都建議使用 `QObject::deleteLater()`.

## copy-on-write

說到 copy-on-write, 請恕我先離題一下, 來談談常見的字串複製行為.

### Mutable String

某些字串內部實作使用的是 mutable string, 即 string 的內容可以改變, 複製時也要複製一份全新的複本. 但如果你其實不會修改這字串, 那完整複製這複本就是很多餘的事了. 以字串的使用情況來說更為極端: 要嘛你會一直修改字串, 要嘛你根本不會修改它. C++ 有 const reference 和 rvalue reference 解決這些問題, 而其他語言則大多採用 immutable string.

### Immutable String

Immutable string 在建立之後內容就不會再被更改了, 所有需要修改內容的操作都是回傳一個新的複本. 很多常見的語言使用的都是 immutable string.

由於內容不變, 複製時可以直接共用同一塊空間, 也不需要考慮同步問題, 可說是非常高效的作法. 但相對來說, 修改字串的成本就變大了, 比方說字串連續串接, 過程中會一直產生立即被丟掉的暫時字串. 因此使用 immutable string 的語言通常也會提供工具讓你先在 buffer 內處理字串內容, 再一次生成結果字串, 例如 Java 的 StringBuilder 或是 Python 的 cStringIO 等.

### Copy On Write

好啦, 扯這麼遠終於又回到 copy-on-write 啦. COW 基本上是在 mutable 和 immutable 之間取平衡點: 在複製時它先共用同一塊記憶體, 等到第一次要修改時才真正地複製一份完整的複本出來. 這看起來似乎解決了所有的問題.

事情當然是不會那麼美好, COW 的其中一個問題是, 所謂*修改*語意在程式語言裡很難精確地表達. 在 C++, 一個可能的做法是把所有 non-const 的成員函式呼叫都視為修改, 這正是 `QSharedDataPointer` 採用的方針.

先談談 Qt 如何透過 `QSharedData` 以及 pimpl idiom 讓你輕易實作 COW. 架構大致上會像這樣:

```cpp
class Employee {
public:
    Employee ();
    // other member functions
private:
    class Data;
    QSharedDataPointer<Data> d;
};

class Employee::Data: public QSharedData {
public:
    Data ();
    Data (const Data &);
    // other member variables
};
```

`QSharedData` 不只提供型別標記, 也提供一個 thread-safe 的計數器. `QSharedDataPointer` 就跟一般的 smart pointer 一樣提供 operator-> 的重載, 只是 non-const 版本會自動呼叫 `QSharedDataPointer::detach()`. `detach()` 的內容大致上像這樣(已簡化):

```cpp
void QSharedDataPointer::detach () {
    if( this->d->rc > 1 ) {
        Data * d = this->d;
        this->d = Type(*d);
        --d->rc;
    }
}
```

Qt 的大部分具有 value 語意的 classes 都實作了 implicit-sharing, 如 QString, QByteArray 以及所有的 template container. 而 gcc 的 std::string 實作也採用 COW.

看起來很完美, 但事情總有例外: 如果物件洩漏了內部的成員, 比方說 container 很常見的 `T & operator []`, 那麼 `detach()` 會被輕易地繞過:

```java
String a = "hello";
Char & c = a[0];
String b = a;
c = 'a';
assert(b == "hello");
```

QString 避免這問題的方法是, 讓 non-const `operator []` 回傳一個特別的物件, 其型別為 `QCharRef`, 在內容修改時為 `QString` 呼叫 `detach()`. 但其他的 template container 就沒辦法如法炮製了, 上述問題依然存在. 相較之下 gcc 的解法就比較優雅一些, 它的計數器多了一種狀態: leaked, 呼叫 non-const `operator []` 之後計數器的狀態會是 leaked, 下次的複製就一定會是完整複製.

如果對 `QSharedDataPointer` 的自動化有疑慮, `QExplicitlySharedDataPointer` 即是為此存在. 它不會為你自動呼叫 `detach()`, 你必須自行分辨使用時機.

## Summary

Qt 原本就設計了兩種回收機制, 全都基於 `QObject` 運作, 並和 multi-thread 與 event-loop 有緊密的結合. 而為了解決各家 compiler 對新的 C++ 標準支援不一的問題, 提供了常見的數種 smart pointers; 當然, 它們與 Qt-style STL 一樣, 用不用隨你.
