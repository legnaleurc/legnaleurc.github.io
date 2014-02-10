<!--
.. title: Garbage Collection (1)
.. slug: garbage-collection-1
.. date: 2012/07/15 21:41:12
.. tags:
.. link:
.. description:
.. type: text
-->

這篇只是介紹比較知名的方法，預計之後還會再發一兩篇討論 C++/Qt 的回收手法。

## Reference Counting

C/C++ 裡最常被使用的方式, 也有部分 GC 演算法會混合 reference counting. 其原理是每個物件都有一個計數器, 記錄目前有多少其他物件指向它. 第一次建立時計數器是 1, 每次複製時計數器加 1, 每少一個物件指向它就減 1, 如果計數器降到 0 就代表此物件不需要存在, 這時它會被刪除. 圖解如下:

![](https://lh4.googleusercontent.com/-NWnj-veDuxc/UALDHo-fpSI/AAAAAAAAAi8/DDOY0h2KqxM/s800/rc1.PNG)

但 reference counting 有一個重大的缺點, 就是在有 cyclic reference 的狀況下, 它無法回收物件. 圖例:

![](https://lh3.googleusercontent.com/-y-Ek_yWpUNM/UALE5Pp89yI/AAAAAAAAAjU/smqBu3rT010/s800/rc3.PNG)

變數 c 若因為某些原因而需要持有 b (比方說要保存 parent-child 關係), 就形成了 cyclic reference, 此時就算變數 a 不再指向它們, 但因為計數器不會歸零, 它們永遠無法被回收. 圖例:

![](https://lh4.googleusercontent.com/-wqBg5DlOsX4/UALE5Nl9CAI/AAAAAAAAAjQ/rfllXgIcrrA/s800/rc2.PNG)

要解決這個問題可以使用 weak reference counting (後述), 但因為它的使用方式與 strong reference counting 不同, programmer 必須要自行檢查是否會造成 cyclic reference, 再視情況改用 weak reference. 然而很多時候 cyclic reference 是很難被發現的, 對 programmer 來說是一項不小的負擔.

## JavaScript

JavaScript 的 closure 常會意外地造成 cyclic reference, 以下是其中一個簡單的例子:

```javascript
window.onload = function () {
    var tmp = document.getElementById('...');
    tmp.onclick = function () {
        // tmp may reference here because closure
    };
};
```

通常在 DOM event handler 中經常會出現這種寫法, 畢竟如果不使用 closure, 寫法會變得很不自然. 所幸目前大部分的 JavaScript engine 都已經使用 mark and sweep (後述), 基本上也不太需要過於擔心這問題 ... 除了早期的 IE 以外(講明一點, 在 Windows XP 上的 IE 版本似乎都還在用 reference counting).

## Mark and Sweep

近代的 GC 有許多都是以 mark and sweep 為主，再混合一點變型(如 Python 也有用上 reference counting). Mark and sweep 的想法也很簡單: 首先走訪目前存在的所有變數, 把碰得到的物件標記起來，此為 mark; 接著檢查所有物件, 把剛剛沒被標記的物件回收, 有被標記的物件清除標記以供下回使用, 此為 sweep. 寫成演算法即:

```python
def mark(variables):
    for v in variables:
        if not v.marked:
            v.marked = True
            mark(v.children)

def sweep(objects):
    for o in objects:
        if o.marked:
            o.marked = False
        else:
            del o
```

由於 mark and sweep 是直接走訪所有的變數及物件, 不像 reference counting 只知道 parent-child 的關係, 因此它不受 cyclic reference 的影響. 但也因此, C/C++ 很難直接原生支援, 因為它經常需要在 memory pool 上再做一些額外處理. 另外，在 mark and sweep 執行時, 為了 thread safety, 通常會先暫停所有的 thread, 這對效能可能是另一項衝擊.

就算有 VM 可以幫 programmer 做資源回收, 但「資源」並不只限於 heap. 對於 sockets, IO devices, shared memory 等外部資源, VM 還是無法完全幫 programmer 監視所有資源的使用, 因此 client 還是有必要手動呼叫回收函式(如 close(), release() 等).
