---
layout: post
title:  leveldb源码分析-8-compaction
date:   2016-12-29 20:00:11
category: "leveldb"
---

# leveldb Compaction
  leveldb Compaction 会在以下场景触发：

* `DB::Open`时检查条件满足时，触发
* `DB::Write`操作里的`MakeRoomForWrite`的条件满足时，触发
* `DB::Get`操作时，在内存中没有命中且Seek文件超过阈值时，触发
* `DB::CompactRange`时，触发

  被触发的compaction都会被`env_->Schedule(&DBImpl::BGWork, this)`调度在bg thread上进行执行，
  默认实现为在bg thread上串行调用`DBImpl::BackgroundCompaction()`。


# DBImpl::BackgroundCompaction()

### memtable compaction
  当memtable超过写缓存大小后，memtable从可变表变成不可变immutable。当触发compaction时，先将
  immutable生成SSTable，然后将SSTable写入文件中，然后将该文件记录到合适的level中。选取指定
  level时，如果SSTable中的key都是新key，则选取最大的level，如果SSTable中的key与高level(最高为
  kMaxMemCompactLevel 2)的文件重叠时，选取低的level，key频繁更新时，新生成的SSTable都存在level-0上
  （尽量不往level-0上放是避免较多的level-0到level-1的compaction）。

  然后将相关元数据更新到VersionEdit上，apply到VersionSet中。apply时会从全部SSTable文件中挑出
  每个level需要保留的文件，再对每个level进行打分(`Version::compaction_score_`)。然后更新MANIFEST
  文件和CURRENT文件，和VersionSet中的current Version。

  然后清除不需要的文件。

### SSTable compaction
  当没有immutable需要compaction时，则进行SSTable的compaction。此时判断compact是否是用户触发的。
  如果是用户触发的则从SSTable中选取用户compact的key的范围。如果是自动触发的compact则调用`VersionSet.PickCompaction()`
  获取需要compact的level和key的范围。`VersionSet.PickCompaction()`根据之前对每个level的打分，选
  择最佳的level，然后挑选出这个level的文件和compact key的范围。这些信息汇总在`Compaction`中。

  然后根据`Compaction`中的信息判断采用TrivialMove还是常规compact。

#### TrivialMove
  当compact level的范围内只有1个文件且与level+1没有重叠则采用`TrivialMove`。因为2层level之间没有
  重叠，则简单的将level的文件移动到level+1层即可。

#### 常规compaction
  不满足`TrivialMove`时则采用常规compaction。调用`DBImpl::DoCompactionWork`进行常规compaction。
  常规compaction会把之前选取到的全部文件构建一个`MergingIterator`来有序遍历这些文件中的key。将
  `kTypeDeletion`类型的sequence最大的key的记录输出到新产生的文件中，其他的记录都被抛弃掉。从而
  完成SSTable文件的合并，当新生成的文件大小到达阈值时，会另起一个文件。

  合并文件完成更新VersionSet中的Version信息。

  在常规compaction时，仍然会检测是否有新的immutable生成，如果有则有限处理immutable compaction。
  然后清除不需要的文件。

  然后，如果是手动触发的compaction，通知手动compaction结果。

### Compaction带来的问题

#### 写放大
  从上面的compaction可以得知，当一个key频繁更新时，每次compaction在该key上都会导致重叠，因此会
  发生一次该key的搬移，导致实际的磁盘写量大于用户调用的写量。

#### 读放大
  leveldb是以文件为单位读取的，当`Get`一个已经被compaction搬移到深层次的key时，会发生多次文件的
  读取，导致实际的磁盘读量大于用户`Get`的调用量。
