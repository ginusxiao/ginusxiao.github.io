# 提纲
[toc]

# 简介
rocksdb的版本相关的数据结构有Version、VersionStorageInfo、VersionBuilder、VersionEdit、SuperVersion和VersionSet。

VersionEdit描述的是版本的变更，其主要操作为AddFile和DeleteFile，分别表示在某个level上增加文件和删除文件，都是版本变更的操作。

VersionBuilder是生成Version的工具，其有两个主要函数：
```
    void Apply(VersionEdit* edit);
    void SaveTo(VersionStorageInfo* vstorage);
```

分别是应用某个VersionEdit和将现在的版本保存到某个VersionStorageInfo。

VersionStorageInfo是Version的信息的存储结构，每一个Version的sst文件信息都保存在VersionStorageInfo。

Version是一个完整的版本。sst文件信息存储在VersionStorageInfo。可以在这个版本上Get数据。

SuperVersion是db的一个完整的版本包含的所有信息，包含当前的Memtable，ImmutableMemTable和一个Version。也就是Version包含的是sst数据信息，SuperVersion包含的是Version的数据和memtable中的数据。

VersionSet是整个db的版本管理，并维护着manifest文件。每个column family的版本单独管理，在ColumnFamilyData这个数据结构里面有current Version。

# Rocksdb之Version
Rocksdb之Version(http://note.youdao.com/noteshare?id=a84f802d06c7591ee5eefbee711158c2&sub=CCB2C76901B149528A443CEE1BD08244)

# Rocksdb之VersionStorageInfo
Rocksdb之VersionStorageInfo(http://note.youdao.com/noteshare?id=6d7e4613e74fcf666b72e4668509ceee&sub=D2814DDFE3C942D4837BC507A8368264)

# Rocksdb之VersionBuilder
Rocksdb之VersionBuilder(http://note.youdao.com/noteshare?id=924e8c4bf91054f855b8dd5a00892c22&sub=68E2C03F50FE47F19788AE0B928C76DE)

# Rocksdb之VersionEdit
Rocksdb之VersionEdit(http://note.youdao.com/noteshare?id=6b2b0b58ace66a648ee8d4cc96b16330&sub=8EFBD05CEB7F48DD8696034600EF2A1A)

# Rocksdb之SuperVersion
Rocksdb之SuperVersion(http://note.youdao.com/noteshare?id=bf7d36b1a030a35ddef483ce859d2973&sub=C8FEBAB2096E4FD6B71C5008CB318AEE)

# Rocksdb之VersionSet
Rocksdb之VersionSet(http://note.youdao.com/noteshare?id=4dc1fd9d86b22646dd68d245952208f3&sub=87981671D3504ABB8BA2E2B4F794407D)


# 参考
[How we keep track of live SST files](https://github.com/facebook/rocksdb/wiki/How-we-keep-track-of-live-SST-files)

[MANIFEST](https://github.com/facebook/rocksdb/wiki/MANIFEST)
