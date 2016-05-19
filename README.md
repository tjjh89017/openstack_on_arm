# OpenStack on ARM

其實這就只是篇心得記錄而已。可能會提到些 OpenStack 在非 x86 上的問題與修改經驗。

基本上來說會去搞 OpenStack on ARM ，主要是因為 jserv 在成大的課程當中需要用到 ARM 的環境，但是並不是能夠有足夠的經費發給個人一組實際的測試環境，剛好在某次跟朋友的聚會上看到了 Scaleway 的 ARM 主機的服務（當時還在 Open Beta 而已），總之就萌生用 OpenStack 加上 ARM 開發板弄 VPS 給修課學生使用的念頭。

當然一開始超不順利的，哪有人瘋了在 OpenStack 支援以外的地方安裝，所以計畫開始了幾個月之後就停滯了，那時候用 Raspberry Pi 1 作為計算的節點。當初已經找到可以用 Docker 來代替 KVM ，但是始終沒有辦法成功將 Container 執行起來。總之因為還有些課業壓力，就把計畫放置了。

直到 jserv 在暑假開始把課程進入 ARMv8 的之後，ARMv8 的開發板取得更為困難，而 QEMU 會用 JIT Compiler 的方式重新調整 ARM 執行方式，也沒有 CPU 快取的機制，所以簡單說就是，雖然可以作為測試用，但是效能數據卻沒有辦法反映實際的結果，所以這個計畫又被開啟了。這次我學乖了，先在 x86 上將 OpenStack 建立起來，然後用 Banana Pi M2 嘗試建立個 Cloud 出來，當然問題也是相當的多，不過發現問題之後，回到 x86 Cloud 上將問題重現再嘗試解決，Debug 速度就快了很多，雖然還是花了些時間，不過基本上 Docker based 的 OpenStack 就成功搞定了！雖然有些效能問題，不過基本上是進入軌道了。

而就在去年（ 2015/11 ），技嘉推出了一張伺服器導向的 ARMv8 主機板（ MP30-AR0 ），剛好因緣際會的借到了這張板，這玩意麻煩的點就不提了，總之我跟朋友花了近 4 個月的時間都在崩潰的跟他奮鬥，最後終於在上面安裝 Linux 成功。這只是其中的一小步而已，因為接下來還得安裝個 OpenStack ，感謝的在於 KVM 幾乎是完全會動的，不用其他處理就會跑了！花了些時間選到了有 OpenStack 支援又不會 CPU Freezing 的發行版（ Fedora 23 AArch64 ）。

果不其然，人生沒有那麼美好的事， `qemu-system-aarch64` 上面是沒有辦法開圖形化模式的，也沒有預設編譯有 `SPICE` 支援，所以在 VNC 跟 SPICE 都不會動的狀態下要把可能在 VM 的網路問題或是開機問題給解了，反正有 console log 就先當作沒差了。運氣不錯， Ubuntu 有幫 ARMv8 準備好 Cloud 映像檔可以用，正當以為可以直接上的時候，發現 VM 開機之後只會 CPU 100% ，但是完全沒有任何執行 Linux 核心的跡象。我以為是什麼環節沒有搞定，所以就重新安裝了整套系統了三次，但是同樣問題就依舊出現。

直到某天我去交大上課，在未來的實驗室晃的時候，學長就來跟我討論一些 OSDI 的問題，當下才發現到一件事， QEMU 預設的 Bootloader 是 BIOS ，但是 ARMv8 的系統上並沒有 BIOS 也沒有 MBR sector 啊！在一番 Google 搜尋之下找到了 Linaro 在安裝 OpenStack 的文件，發現可以直接指定 kernel 跟 initrd 給 QEMU ，才終於讓第一個 VM 成功開啟。

也因為發現了 Linaro 的文件，看到 Linaro.cloud 的專案，大概 3 月初的時候發布消息的，而我大概在 5 月初完成 OpenStack on ARM  的雛型，雖然之前就有看到 Linaro 有相關的成果，不過這一次是 Open Beta ，而且也預計在北美、歐洲、中國等地都提供 ARMv8 Cloud 的環境給開發者們測試使用。看到當然就得跟他們聯繫下啦，後來 Linao 中國的員工來跟我聯繫，說明他們是世界上第一個的 ARMv8 Cloud （那這樣不知道我有沒有辦法宣稱我是第二個），總之就是簡單介紹下我已經完成的東西，雖然還有待加強啦，不過我跟朋友用少量資本完成雛型應該也可以算是一大成就。

那開始跟 jserv 進行 Cloud 的 Close Beta ，找了幾位在資安界打比賽的朋友來幫忙測試，為什麼找他們呢？因為在實驗室一直聽到學長在說 QEMU 有多慢就有多慢，偏偏國外比賽越來越多非 x86 的環境，所以想當然爾的就請他們來當第一批玩家。當然沒什麼好說的，效果就是比 QEMU 好上數倍，學長似乎也在這機器上面開始他的碩論測試，不知道會不會列入感謝名單之中，覺得期待。

不過學長也很快就提出了些新需求，希望可以也支援 ARMv7 ，聽起來挺簡單的，因為 ARMv8 CPU 大部份都可以直接支援 ARMv7 指令集，所以我想應該只要把 Kernel 、 Initrd 、 Rootfs 準備好應該就沒有問題了吧。不過出問題是必然的，立刻就發現跟之前的 CPU 100% ，但是情況不一樣， 是已經進入的 Linux 核心之後才 CPU 100% 滿載，用之前研究 Linux 核心的經驗簡單排除幾個問題之後，也看到有其他國外網友成功用 virt-manager 執行 ARMv7 VM 成功，但是自己一直都沒有辦法用 QEMU 重現對方的結果，又進入了個死胡同。嘛，就是這樣嘛，人生就是充滿著麻煩，才感受到活著。過了一週的今天（ 2016/5/19 ），總統就職的前一天，用 QEMU 跟 gdb 想研究看看卡在核心的哪個部分時發現， `qemu-system-aarch64` 在執行的時候立刻就跳進了 Undefined Interrupt Handler ，才忽然意識到 ARMv8 跟 ARMv7 的指令集 Opcode 是互不相容的，雖然 CPU 可以正確判斷並執行，但是 `qemu-system-aarch64` 沒有辦法直接決定使用的指定集，而必須要在 `-cpu` 的參數中將 AArch64 的模式關閉才能（`-cpu host,aarch64=off`），知道 QEMU 怎麼操作了，然後研究個 Libvirt Domain XML 該怎麼寫，似乎加個屬性到 `<type>` 裡面就行（`<type arch=armv7l ...>`）。問題來了， OpenStack Nova 並沒有辦法提供這個屬性給 Libvirt ，所以基本上我得去幫 Libvirt Driver 修個 Code 、上個 Patch 了，但是現有的唯一 ARMv8 伺服器在 Close Beta 中，所以基本上有想法沒辦法實踐，就只好先把問題擺著等 Close Beta 結束，或是有新的環境可以使用才開始了。

而就在睡不著覺的現在（ 2016/5/20 02:47 ）把心得分享給大家，如果有興趣可以關注 [DozenCloud 計畫](http://dozencloud.org)以及 [DozenCloud Github](http://github.com/dozencloud/) 裡面會在之後更新 OpenStack on ARM 相關的文件，雖然有些已經很久沒有修改動作了，不過基本上計畫一直都在執行當中，只是文件會比較慢才生出來就是了。

特別感謝眾多朋友的幫忙