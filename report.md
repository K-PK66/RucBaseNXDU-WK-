# Rucbase 实验一（存储管理）实验报告

## 1 缓冲池管理器
## 1.1 磁盘存储管理器
任务要求补全`DiskManager`类，负责读写指定页面、分配页面编号及对文件进行操作。

根据题中所提供的`DiskManager`类的各个接口，可通过`storage/disk_manager.cpp`来实现`storage/disk_manager.h`中对其的定义，从而使`DiskManager`可以实现整个项目的统一文件管理。

### 1.1.1 读写页面
* `DiskManager::write_page`
  
  ```cpp
  void DiskManager::write_page(int fd, page_id_t page_no, const char *offset, int num_bytes) {
      if( fd2path_.find(fd) == fd2path_.end() ) {
          throw FileNotOpenError(fd);
      }
      lseek(fd, page_no * PAGE_SIZE, SEEK_SET);
      if( write(fd, (void *)offset, num_bytes) != num_bytes ) {
          throw UnixError();
      }
  }
  ```

- `DiskManager::read_page`
  
  ```cpp
  void DiskManager::read_page(int fd, page_id_t page_no, char *offset, int num_bytes) {
      if( fd2path_.find(fd) == fd2path_.end() ) {
          throw FileNotOpenError(fd);
      }
      lseek(fd, page_no * PAGE_SIZE, SEEK_SET);
      if( read(fd, (void *)offset, num_bytes) < 0 ) {
          throw UnixError();
      }
  }
  ```

### 1.1.2 分配页面编号
- `DiskManager::AllocatePage`
  
  ```cpp
  page_id_t DiskManager::AllocatePage(int fd) {
      return fd2pageno_[fd] ++; //
  }
  ```

### 1.1.3 文件操作

- `DiskManager::is_file`
  
  ```cpp
  bool DiskManager::is_file(const std::string &path) {
      struct stat st;
      return stat(path.c_str(), &st) == 0 && (S_IFREG ==(st.st_mode & S_IFMT));
  }
  ```

- `DiskManager::create_file`
  
  ```cpp
  void DiskManager::create_file(const std::string &path) {
      if(!is_file(path)) {
          auto fd = open( path.c_str(), O_CREAT | O_RDWR, 0x1ff );
          close(fd);
      } else {
          throw FileExistsError(path);
      }
  }
  ```

- `DiskManager::close_file`
  
  ```cpp
  void DiskManager::close_file(int fd) {
      auto it = fd2path_.find(fd);
      if( it != fd2path_.end() ) {
          close(fd);
          path2fd_.erase( path2fd_.find((*it).second) );
          fd2path_.erase(it);
      } else {
          throw FileNotOpenError(fd);
      }
  }
  ```

* `DiskManager::destroy_file`
  
  ```cpp
  void DiskManager::destroy_file(const std::string &path) {
      if( !is_file(path) ) {
          throw FileNotFoundError(path);
      }
      if( path2fd_.find(path) == path2fd_.end() ) {
          if( unlink(path.c_str()) < 0 ) {
              throw UnixError();
          }
      } else {
          throw FileNotClosedError(path);
      }
  }
  ```
## 1.2 缓冲池替换策略

注意到`Replacer`类可以实现`storage/buffer_pool_manager.h`中定义的`BufferPoolManager`类的缓存替换算法，可以通过修改`replacer/lru_replacer.cpp`实现修改`replacer/lru_replacer.h`的各个`LRUReplacer`类的接口；后者继承自`replacer/replacer.h`中定义的`Replacer`类。

通过代码实现如下：

- `LRUReplacer::Victim`
  
  ```cpp
  bool LRUReplacer::Victim(frame_id_t *frame_id) {
      // C++17 std::scoped_lock
      // 它能够避免死锁发生，其构造函数能够自动进行上锁操作，析构函数会对互斥量进行解锁操作，保证线程安全。
      std::scoped_lock lock{latch_};
  
      if( LRUlist_.size() == 0 ) return false;
  
      LRUhash_.erase( LRUhash_.find( (*frame_id) = LRUlist_.back() ) );
      LRUlist_.pop_back();
      return true;
  }
  ```

- `LRUReplacer::Pin`
  
  ```cpp
  void LRUReplacer::Pin(frame_id_t frame_id) {
      std::scoped_lock lock{latch_};
  
      auto it = LRUhash_.find(frame_id);
      if( it != LRUhash_.end() ) {
          LRUlist_.erase( (*it).second );
          LRUhash_.erase( it ); 
  ```

- `LRUReplacer::Unpin`
  
  ```cpp
  void LRUReplacer::Pin(frame_id_t frame_id) {
      std::scoped_lock lock{latch_};
  
      auto it = LRUhash_.find(frame_id);
      if( it != LRUhash_.end() ) {
          LRUlist_.erase( (*it).second );
          LRUhash_.erase( it ); 
      }
  }
  ```

### 1.3 缓冲池管理器

可通过修改文件`storage/buffer_pool_manager.cpp`来实现对`storage/buffer_pool_manager.h`中定义的`BufferPoolManager`类的接口进行修改。后者可以给`record/rm_file_handle.h`中定义的`RmFileHandle`记录文件管理类和` index/ix_index_handle.h`中定义的 `IxIndexHandle`索引文件管理类提供统一的文件和缓存管理功能。

通过代码实现如下：

- `BufferPoolManager::FindVictimPage`
  
  ```cpp
  bool BufferPoolManager::FindVictimPage(frame_id_t *frame_id) {
      if( free_list_.size() != 0 ) {
          (*frame_id) = free_list_.front();
          free_list_.pop_front();
          //std::cout << "alloc new frame " << *frame_id << "]]]]]\n";
          return true;
      }
      return replacer_ -> Victim( frame_id );
  }
  ```

- `BufferPoolManager::UpdatePage`
  
  ```cpp
  void BufferPoolManager::UpdatePage(PageId new_page_id, frame_id_t frame_id) {
      Page *page = &pages_[frame_id];
  
      //std::cout << "[Buffer] Update: frame " << frame_id << " (" << page -> id_.fd << ", " << page -> id_.page_no <<
      //    ") -> (" << new_page_id.fd << ", " << new_page_id.page_no << ")\n";
  
      if( page -> IsDirty() ) {
          disk_manager_ -> write_page(page -> GetPageId().fd, page -> GetPageId().page_no, page -> GetData(), PAGE_SIZE);
          page->is_dirty_ = false;
      }
      page -> ResetMemory();
  
      auto it = page_table_.find(page -> id_);
      if( it != page_table_.end() ) page_table_.erase(it);
      page_table_[page -> id_ = new_page_id] = frame_id;
  
      if( page -> id_ .page_no != INVALID_FRAME_ID ) {
          disk_manager_ -> read_page(page -> GetPageId().fd, page -> GetPageId().page_no, page -> GetData(), PAGE_SIZE);
      }
  }
  ```
特别地，这里修改了`storage/buffer_pool_manager.h`中的一部分默认定义，便于写入脏页并加载替换后的新页。

- `BufferPoolManager::FetchPage`
  
  ```cpp
  Page *BufferPoolManager::FetchPage(PageId page_id) {
      std::scoped_lock lock{latch_};
  
      frame_id_t frame_id;
  
      if( page_table_.find(page_id) != page_table_.end() ) {
          //std::cout << "[Buffer] Cache hit! fetch {" << page_id.fd << ", " << page_id.page_no << "}\n";
          frame_id = page_table_[page_id];
      } else {
          //std::cout << "[Buffer] Cache miss! Try to fetch page (" << page_id.fd << ", " << page_id.page_no << ") from disk\n"; 
          if( !FindVictimPage(&frame_id) ) return nullptr;
          UpdatePage(page_id, frame_id);
      }
  
      replacer_ -> Pin(frame_id);
      pages_[frame_id].pin_count_ ++;
  
      return &pages_[frame_id];
  }
  ```

- `BufferPoolManager::NewPage`
  
  ```cpp
  Page *BufferPoolManager::NewPage(PageId *page_id) {
      std::scoped_lock lock{latch_};
  
      frame_id_t frame_id;
      if( !FindVictimPage(&frame_id) ) return nullptr;
  
      page_id -> page_no = disk_manager_ -> AllocatePage( page_id -> fd );
  
      UpdatePage(*page_id, frame_id);
  
      replacer_ -> Pin(frame_id);
      pages_[frame_id].pin_count_ ++;
  
      return &pages_[frame_id];
  }
  ```

- `BufferPoolManager::DeletePage`
  
  ```cpp
  bool BufferPoolManager::DeletePage(PageId page_id) {
      std::scoped_lock lock{latch_};
  
      if( page_table_.find(page_id) == page_table_.end() ) return true;
      frame_id_t frame_id = page_table_[page_id];
  
      Page *page = &pages_[frame_id];
  
      if( page -> pin_count_ > 0 ) return false;
  
      disk_manager_->DeallocatePage(page_id.page_no);
  
      page_id.page_no =  INVALID_PAGE_ID;
      UpdatePage(page_id, frame_id);
      free_list_.push_back(frame_id);
  
      return true;
  }
  ```

## 2 记录管理器

### 2.1 记录操作
文件`record/rm_file_handle.cpp`可以用于`record/rm_file_handle.h`中实现记录文件管理类`RmFileHandle`。

基于上述分析，可通过如下代码实现功能：

* `RmFileHandle::create_new_page_handle`
  
  ```cpp
  RmPageHandle RmFileHandle::create_new_page_handle() {
      PageId pageid = {fd_, INVALID_FRAME_ID};
      Page* page = buffer_pool_manager_ -> NewPage(&pageid);
  
      auto pageHandler = RmPageHandle(&file_hdr_, page);
  
      if( page != nullptr ) {
          file_hdr_.num_pages ++;
          file_hdr_.first_free_page_no = pageid.page_no;
  
          pageHandler.page_hdr -> next_free_page_no = RM_NO_PAGE;
          pageHandler.page_hdr -> num_records = 0;
          Bitmap::init(pageHandler.bitmap, file_hdr_.bitmap_size);
      }
  
      return pageHandler;
  }
  ```

- `RmFileHandle::fetch_page_handle`
  
  ```cpp
  RmPageHandle RmFileHandle::fetch_page_handle(int page_no) const {
      if( page_no >= file_hdr_.num_pages ) {
          throw PageNotExistError(disk_manager_ -> GetFileName(fd_), page_no);
      }
      return RmPageHandle(&file_hdr_, buffer_pool_manager_ -> FetchPage( {fd_, page_no} ));
  }
  ```


- `RmFileHandle::create_page_handle` 
  
  ```cpp
  RmPageHandle RmFileHandle::create_page_handle() {
      if(file_hdr_.first_free_page_no == RM_NO_PAGE) {
          return create_new_page_handle();
      }
      return fetch_page_handle(file_hdr_.first_free_page_no);
  }
  ```

- `RmFileHandle::release_page_handle`
  
  ```cpp
  void RmFileHandle::release_page_handle(RmPageHandle &page_handle) {
      page_handle.page_hdr -> next_free_page_no = file_hdr_.first_free_page_no;
      file_hdr_.first_free_page_no = page_handle.page -> GetPageId().page_no;
  }
  ```

- `RmFileHandle::get_record`
  
  ```cpp
  std::unique_ptr<RmRecord> RmFileHandle::get_record(const Rid &rid, Context *context) const {
      auto record = std::make_unique<RmRecord>(file_hdr_.record_size);
      auto pageHandler = fetch_page_handle(rid.page_no);
  
      if( !Bitmap::is_set(pageHandler.bitmap, rid.slot_no) ) {
          throw RecordNotFoundError(rid.page_no, rid.slot_no);
      }
  
      record -> size = file_hdr_.record_size;
      memcpy( record -> data, pageHandler.get_slot(rid.slot_no), file_hdr_.record_size );
      return record;
  }
  ```

- `RmFileHandle::insert_record`
  
  ```cpp
  Rid RmFileHandle::insert_record(char *buf, Context *context) {
      auto pageHandler = create_page_handle();
      int slot_no = Bitmap::first_bit(false, pageHandler.bitmap, file_hdr_.num_records_per_page);
  
      memcpy( pageHandler.get_slot(slot_no), buf, file_hdr_.record_size );
      Bitmap::set(pageHandler.bitmap, slot_no);
  
      if( ++ pageHandler.page_hdr -> num_records == file_hdr_.num_records_per_page ) {
          file_hdr_.first_free_page_no = pageHandler.page_hdr -> next_free_page_no;
      }
  
      //std::cout << "insert recoder {" << pageHandler.page -> GetPageId().page_no << " " << slot_no << "}\n";
  
      return Rid{pageHandler.page -> GetPageId().page_no, slot_no};
  }
  ```

- `RmFileHandle::delete_record`
  
  ```cpp
  void RmFileHandle::delete_record(const Rid &rid, Context *context) {
  
      //std::cout << "delete recoder {" << rid.page_no << " " << rid.slot_no << "}\n";
  
      auto pageHandler = fetch_page_handle(rid.page_no);
  
      if( !Bitmap::is_set(pageHandler.bitmap, rid.slot_no) ) {
          throw RecordNotFoundError(rid.page_no, rid.slot_no);
      }
  
      Bitmap::reset(pageHandler.bitmap, rid.slot_no);
      if( (pageHandler.page_hdr -> num_records ) -- == file_hdr_.num_records_per_page ) {
          release_page_handle(pageHandler);
      }
  }
  ```

- `RmFileHandle::update_record`
  
  ```cpp
  void RmFileHandle::update_record(const Rid &rid, char *buf, Context *context) {
      //std::cout << "update recoder {" << rid.page_no << " " << rid.slot_no << "}\n";
      auto pageHandler = fetch_page_handle(rid.page_no);
  
      if( !Bitmap::is_set(pageHandler.bitmap, rid.slot_no) ) {
          throw RecordNotFoundError(rid.page_no, rid.slot_no);
      }
      memcpy( pageHandler.get_slot(rid.slot_no), buf, file_hdr_.record_size );
  }
  ```

### 2.2 记录迭代器

注意到`record/rm_scan.cpp`可实现` record/rm_scan.h`中定义的`RmScan`迭代器类，后者可以用于遍历`RmFileHandle`对象内部的记录，故可通过如下代码实现：

- `RmScan::next`
  
  ```cpp
  void RmScan::next() {
      while( rid_.page_no < file_handle_ -> file_hdr_.num_pages ) {
          auto page_handler = file_handle_ -> fetch_page_handle(rid_.page_no);
          rid_.slot_no = Bitmap::next_bit( true, page_handler.bitmap, file_handle_->file_hdr_.num_records_per_page, rid_.slot_no );
  
          if( rid_.slot_no < file_handle_ -> file_hdr_.num_records_per_page) {
              //std::cout << "{" << rid_.page_no << " " << rid_.slot_no << "}" << file_handle_->file_hdr_.num_pages << " " 
              //    << file_handle_->file_hdr_.num_records_per_page << " " << file_handle_->file_hdr_.bitmap_size <<"\n";
              return;
          }
          rid_ = Rid{ rid_.page_no + 1, -1 };
          if( rid_.page_no >= file_handle_ -> file_hdr_.num_pages ) {
              rid_ = Rid{RM_NO_PAGE, -1};
              break;
          }
      }
  }
  ```


## 3 测试结果

### 3.1 测试命令

在项目的根目录下编写一个`Makefile`，用于调用Rucbase Lab的单元测试。

```makefile
# ./Makefile
copy:
    cd build; \
    cmake .. -DCMAKE_BUILD_TYPE=Debug

t1_1: copy
    cd build; \
    make disk_manager_test; \
    ./bin/disk_manager_test

t1_2: copy
    cd build; \
    make lru_replacer_test; \
    ./bin/lru_replacer_test

t1_3: copy
    cd build; \
    make buffer_pool_manager_test; \
    ./bin/buffer_pool_manager_test

t1_4: copy
    cd build; \
    make rm_gtest; \
    ./bin/rm_gtest
```
分别用如下代码测试上述各函数，分别得到对应结果：
* `make t1_1`
* `make t1_2`
* `make t1_3`
* `make t1_4`
