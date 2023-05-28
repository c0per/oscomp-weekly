# 内存管理、Mmap、Lazy load

陈嘉钰

---

# Maturin
##

```rust
pub struct MemorySet {
    area_map: RangeActionMap<VmArea>,
    pub pt: Box<PageTable>,
}
pub struct VmArea {
    pub(super) start: VirtAddr,
    pub(super) end: VirtAddr,
    pub(super) flags: PTEFlags,
    pub(super) pma: Arc<Mutex<dyn PmArea>>,
}
pub struct PmAreaLazy {
    frames: Vec<Option<Frame>>, // RAII for physical page
    backend: Option<BackEndFile>,
}
pub struct BackEndFile {
    file: Arc<dyn File>,
    offset: usize,
    policy: SyncPolicy,
}
```

---

# Kerla
##

```rust
pub struct Vm {
    page_table: PageTable,
    vm_areas: Vec<VmArea>,
    valloc_next: UserVAddr,
}
pub enum VmAreaType {
    Anonymous,
    File {
        file: Arc<dyn FileLike>,
        offset: usize,
        file_size: usize,
    },
}
pub struct VmArea {
    start: UserVAddr,
    len: usize,
    area_type: VmAreaType,
}
```

---

# 单页的RAII struct
##

ArceOS中原有的`GlobalPage`代表了连续的若干个物理页。

然而在一段虚拟内存（`MapArea`）中我们希望独立地控制每一页是否分配了实际的物理页。

```rust
pub struct PhysPage {
    pub start_vaddr: VirtAddr, // phys addr + offset
}
impl Drop for PhysPage {
    fn drop(&mut self) {
        global_allocator().dealloc_pages(self.start_vaddr.into(), 1);
    }
}
```

---

# 代表虚拟地址段的`MapArea`
##

`MapArea`对应maturin中的`VmArea`，代表一段虚拟地址，被`MemorySet`所有。

```rust
pub struct MapArea {
    pub pages: Vec<Option<PhysPage>>,
    pub vaddr: VirtAddr,
    pub flags: MappingFlags,
    pub backend: Option<MemBackend>,
}
impl Drop for MapArea {
    fn drop(&mut self) {
        self.pages
            .iter()
            .enumerate()
            .filter(|(_, page)| page.is_some())
            .for_each(|(page_index, _)| self.sync_page_with_backend(page_index))
    }
}
```

在回收时，将有file backend的段写入文件中。

`PmArea`目前只实现为`Vec<Option<PhysPage>>`，提供基础的功能。

---

# 向`MemorySet`中添加新的`MapArea`
##

是否有初始数据？是否有file backend？将内存段分为 4 类。

- 对于有初始数据的段，连续分配所有`PhysPage`，修改页表后写入数据。

- 对于没有初始数据的段，不分配`PhysPage`，在页表中写入错误的页表项。

---

# 向`MemorySet`中添加新的`MapArea`
## 普通段

```rust
pub fn new_alloc(
    start: VirtAddr,
    num_pages: usize,
    flags: MappingFlags,
    backend: Option<MemBackend>,
    page_table: &mut PageTable,
) -> Self {
    let pages = PhysPage::alloc_contiguous(num_pages, PAGE_SIZE_4K).unwrap();

    let _ = page_table
        .map_region(
            start,
            virt_to_phys(pages[0].as_ref().unwrap().start_vaddr),
            num_pages * PAGE_SIZE_4K,
            flags,
            false,
        )
        .unwrap();

    Self { pages, vaddr: start, flags, backend }
}
```

---

# 向`MemorySet`中添加新的`MapArea`
## Lazy 段

```rust
pub fn new_lazy(
    start: VirtAddr,
    num_pages: usize,
    flags: MappingFlags,
    backend: Option<MemBackend>,
    page_table: &mut PageTable,
) -> Self {
    let mut pages = Vec::with_capacity(num_pages);
    for _ in 0..num_pages {
        pages.push(None);
    }

    let _ = page_table
        .map_fault_region(start, num_pages * PAGE_SIZE_4K)
        .unwrap();

    Self { pages, vaddr: start, flags, backend }
}
```

---

### mmap

1. 对于相交的原有的段，修改段的大小，可能会将段分为两个
2. 创建新的段，无初始数据，可能有文件backend

### munmap

1. 与mmap中的1类似，调用`split_for_area()` 删除一定区间中的段

```rust
fn split_for_area(&mut self, start: VirtAddr, size: usize) {
    let end = start + size;
    let overlapped_area = self.owned_mem.drain_filter(|area| area.overlap_with(start, end)).collect::<Vec<_>>();
    for area in overlapped_area {
        let _ = self.page_table.unmap_region(area.vaddr, area.size()).unwrap();
        if area_start < start {
            self.new_region(
                area_start.into(),
                start - area_start,
                area.flags,
                Some(&area_data[..start - area_start]),
                area.backend.as_ref().map(|b| b.clone()),
            );
        }
        if end < area_end { /* ... */ }
    }
}
```

---

# `MemBackend` 的 clone 问题
## object safety

在一个具有backend的内存段中，mmap其内部一段时，需要clone原有的`MemBackend`，分别得到两个`offset`不同（指向的File object可以相同）的`MemBackend`作为新拆除的两个子段的backend。

Maturin中，`offset`与`Arc<dyn File>`是两个单独的项，分别对其clone即可。ArceOS中`axfs::api::File`在内部保存了`offset`，而`MemBackend`使用`Arc<SpinNoIrq<dyn FileExt>>`保存文件对象。

这使得我们必须clone`axfs::api::File`本身，但rust不能直接对一个trait object使用clone。目前考虑到有3种解决方法：

- 令`FileExt: AsAny`，通过downcast获得实际的`File`类型后clone。

- 在`MemBackend`中额外维护一个`offset`，每次read/write前手动seek到正确的位置。

- 在`FileExt` trait中添加`clone_file()`方法，仅提供`File`的实现。

---

# 处理 page fault

1. 通过`stval`获取需访问的虚拟地址
2. 在当前进程的`MemorySet`中找到包含此地址的`MapArea`
3. 分配物理页，修改页表
4. 写入数据，有backend则调用其read，否则用0填满

```rust
pub fn handle_page_fault(&mut self, addr: VirtAddr, flags: MappingFlags) {
    match self
        .owned_mem
        .iter_mut()
        .find(|area| area.vaddr <= addr && addr < area.end_va())
    {
        Some(area) => area.handle_page_fault(addr, flags, &mut self.page_table),
        None => panic!("unhandled page fault"),
    }
}
```

---

# `VirtIOHal::share`
##

share 获得一段buffer的物理地址，这段物理地址将与设备（virtio block device）共享，由设备负责将内容写入。

Lazy段中，多次page fault分配的物理内存可能不连续。使得用户传入的buffer在物理地址上可能不连续。

与学长讨论后：分段读取

---

# `axfs::api::File::read()`
##

这个`read()`函数不会读取全部的内容（读入当前cluster中的剩余文件内容），需不断调用制止无法读入。

---

# have done

- 临时添加一个fstat，以通过初赛中的mmap，munmap测例

- 调了一半动态加载，发现并修复了上述`read()`的问题

# todo

- 先运行起动态加载程序

- 页面的dirty标记

- 内核态中断，处理用户buffer可能出现page fault的问题

- 独立`VmArea`与`PmArea`，使得多个`VmArea`可以共享同一个`PmArea`（clone/fork）
