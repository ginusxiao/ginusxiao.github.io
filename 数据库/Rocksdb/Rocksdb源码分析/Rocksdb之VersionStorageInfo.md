# 提纲
[toc]

# 数据结构
## VersionStorageInfo
```
class VersionStorageInfo {
 private:
  const InternalKeyComparator* internal_comparator_;
  const Comparator* user_comparator_;
  /*总的层级数*/
  int num_levels_;            // Number of levels
  /*非空的最大层级（所有比该数值大的层级都一定是空的）*/
  int num_non_empty_levels_;  // Number of levels. Any level larger than it
                              // is guaranteed to be empty.
  // Per-level max bytes
  /*每一个层级的最大容量*/
  std::vector<uint64_t> level_max_bytes_;

  // A short brief metadata of files per level
  /*LevelFilesBrief中记录了某一层级中所有文件的{FileDescriptor, smallest key, largest key}*/
  autovector<rocksdb::LevelFilesBrief> level_files_brief_;
  /* FileIndexer用于优化Rocksdb索引查找，中关于FileIndexer，请参考参考文件中的
   * “【Rocksdb实现分析及优化】FileIndexer”
   */
  FileIndexer file_indexer_;
  /*内存分配器*/
  Arena arena_;  // Used to allocate space for file_levels_

  /*Compaction方式*/
  CompactionStyle compaction_style_;

  // List of files per level, files in each level are arranged
  // in increasing order of keys
  /* 每一层的文件集合的数组，数组中每一个元素是std::vector<FileMetaData*>类型，表示
   * 某一层中文件集合，每一层中的文件以key的升序排列
   */
  std::vector<FileMetaData*>* files_;

  // Level that L0 data should be compacted to. All levels < base_level_ should
  // be empty. -1 if it is not level-compaction so it's not applicable.
  /*Level-0的数据将被合并到的层级，所有小于base_level_的层级都应该是空的*/
  int base_level_;

  // A list for the same set of files that are stored in files_,
  // but files in each level are now sorted based on file
  // size. The file with the largest size is at the front.
  // This vector stores the index of the file from files_.
  /* files_by_compaction_pri_中的文件集合跟files_中的文件集合一样，只不过这里存放
   * 的是files_中的文件在std::vector<FileMetaData*>中的索引编号，而且这里以文件
   * 大小排序存放
   */
  std::vector<std::vector<int>> files_by_compaction_pri_;

  // If true, means that files in L0 have keys with non overlapping ranges
  /*Level-0中的所有文件是否具有交叠的key区间*/
  bool level0_non_overlapping_;

  // An index into files_by_compaction_pri_ that specifies the first
  // file that is not yet compacted
  /*下一个参与合并的文件在files_by_compaction_pri_中的索引号*/
  std::vector<int> next_file_to_compact_by_size_;

  // Only the first few entries of files_by_compaction_pri_ are sorted.
  // There is no need to sort all the files because it is likely
  // that on a running system, we need to look at only the first
  // few largest files because a new version is created every few
  // seconds/minutes (because of concurrent compactions).
  /*只需要排序files_by_compaction_pri_中的前number_of_files_to_sort_个文件*/
  static const size_t number_of_files_to_sort_ = 50;

  // This vector contains list of files marked for compaction and also not
  // currently being compacted. It is protected by DB mutex. It is calculated in
  // ComputeCompactionScore()
  /*等待排序的文件集合*/
  autovector<std::pair<int, FileMetaData*>> files_marked_for_compaction_;

  // Level that should be compacted next and its compaction score.
  // Score < 1 means compaction is not strictly needed.  These fields
  // are initialized by Finalize().
  // The most critical level to be compacted is listed first
  // These are used to pick the best compaction level
  /*每一个层级的compaction score（打分）*/
  std::vector<double> compaction_score_;
  /*下一个参与compaction的层级（最应执行compaction的层级在前）*/
  std::vector<int> compaction_level_;
  int l0_delay_trigger_count_ = 0;  // Count used to trigger slow down and stop
                                    // for number of L0 files.

  // the following are the sampled temporary stats.
  /*下面是抽样的统计信息*/
  // the current accumulated size of sampled files.
  uint64_t accumulated_file_size_;
  // the current accumulated size of all raw keys based on the sampled files.
  uint64_t accumulated_raw_key_size_;
  // the current accumulated size of all raw keys based on the sampled files.
  uint64_t accumulated_raw_value_size_;
  // total number of non-deletion entries
  uint64_t accumulated_num_non_deletions_;
  // total number of deletion entries
  uint64_t accumulated_num_deletions_;
  // current number of non_deletion entries
  uint64_t current_num_non_deletions_;
  // current number of delection entries
  uint64_t current_num_deletions_;
  // current number of file samples
  uint64_t current_num_samples_;
  // Estimated bytes needed to be compacted until all levels' size is down to
  // target sizes.
  uint64_t estimated_compaction_needed_bytes_;

  bool finalized_;

  // If set to true, we will run consistency checks even if RocksDB
  // is compiled in release mode
  bool force_consistency_checks_;

  friend class Version;
  friend class VersionSet;
  // No copying allowed
  VersionStorageInfo(const VersionStorageInfo&) = delete;
  void operator=(const VersionStorageInfo&) = delete;
};
```

# 实现
## VersionStorageInfo::AddFile
VersionStorageInfo::AddFile()将文件添加到某个层级中，添加过程中会检查当前添加的文件和它的前一个文件的key之间的大小关系是否是升序的且不重叠的。
```
void VersionStorageInfo::AddFile(int level, FileMetaData* f, Logger* info_log) {
  /*获取level这一层级的文件集合*/
  auto* level_files = &files_[level];
  // Must not overlap
#ifndef NDEBUG
  if (level > 0 && !level_files->empty() &&
      internal_comparator_->Compare(
          (*level_files)[level_files->size() - 1]->largest, f->smallest) >= 0) {
    /* 因为向VersionStorageInfo中添加文件的时候都是排好序后添加的，所以即将添加的文件
     * f的smallest key必须比VersionStorageInfo中当前的最后一个文件的largest key要大，
     * (*level_files)[level_files->size() - 1]就表示VersionStorageInfo中的最后一个文件
     */
    auto* f2 = (*level_files)[level_files->size() - 1];
    if (info_log != nullptr) {
      Error(info_log, "Adding new file %" PRIu64
                      " range (%s, %s) to level %d but overlapping "
                      "with existing file %" PRIu64 " %s %s",
            f->fd.GetNumber(), f->smallest.DebugString(true).c_str(),
            f->largest.DebugString(true).c_str(), level, f2->fd.GetNumber(),
            f2->smallest.DebugString(true).c_str(),
            f2->largest.DebugString(true).c_str());
      LogFlush(info_log);
    }
    
    assert(false);
  }
#endif

  /*增加添加的文件f的引用计数*/
  f->refs++;
  /*添加到level这一层级的文件集合中*/
  level_files->push_back(f);
}
```

## VersionStorageInfo::RemoveCurrentStats
VersionStorageInfo::RemoveCurrentStats()更新由于删除文件导致的统计信息改变。
```
void VersionStorageInfo::RemoveCurrentStats(FileMetaData* file_meta) {
  if (file_meta->init_stats_from_file) {
    current_num_non_deletions_ -=
        file_meta->num_entries - file_meta->num_deletions;
    current_num_deletions_ -= file_meta->num_deletions;
    current_num_samples_--;
  }
}
```
