---
title: 手撕RocksDB-1
date: 2018-07-03 17:01:34
tags: RocksDB-代码分析
categories: Tech
---

# 手撕RocksDB（1）
## RocksDB的put操作

<!-- more -->

put函数有两个，分别是
```
virtual Status Put(const WriteOptions& options,
                     ColumnFamilyHandle* column_family, const Slice& key,
                     const Slice& value) = 0;
```
以及
```
virtual Status Put(const WriteOptions& options, const Slice& key,
                     const Slice& value) {
    return Put(options, DefaultColumnFamily(), key, value);
  }
```
可以看待下面的put是封装的上面的put，值得注意的是与leveldb不同，rocksdb的put最基本的实现是要指定一个column family的

### put实现
首先声明一个WriteBatch结构，然后调用WriteBatch的put接口将kv数据以及column family数据写入batch，最后调用Write函数进行写入
```
Status DB::Put(const WriteOptions& opt, ColumnFamilyHandle* column_family,
               const Slice& key, const Slice& value) {
  // Pre-allocate size of write batch conservatively.
  // 8 bytes are taken by header, 4 bytes for count, 1 byte for type,
  // and we allocate 11 extra bytes for key length, as well as value length.
  WriteBatch batch(key.size() + value.size() + 24);
  Status s = batch.Put(column_family, key, value);
  if (!s.ok()) {
    return s;
  }
  return Write(opt, &batch);
}
```


### write实现
write通过封装WriteImpl函数实现，所以直接看WriteImpl，首先是排除一系列错误或者不支持的情况
```
  if (my_batch == nullptr) {
    return Status::Corruption("Batch is nullptr!");
  }
  if (write_options.sync && write_options.disableWAL) {
    return Status::InvalidArgument("Sync writes has to enable WAL.");
  }
  if (two_write_queues_ && immutable_db_options_.enable_pipelined_write) {
    return Status::NotSupported(
        "pipelined_writes is not compatible with concurrent prepares");
  }
  if (seq_per_batch_ && immutable_db_options_.enable_pipelined_write) {
    // TODO(yiwu): update pipeline write with seq_per_batch and batch_cnt
    return Status::NotSupported(
        "pipelined_writes is not compatible with seq_per_batch");
  }
```
然后是一系列的特殊情况，需要调用到其他的write实现
```
if (write_options.low_pri) {
    status = ThrottleLowPriWritesIfNeeded(write_options, my_batch);
    if (!status.ok()) {
      return status;
    }
  }

  if (two_write_queues_ && disable_memtable) {//禁用memtable的情况
    return WriteImplWALOnly(write_options, my_batch, callback, log_used,
                            log_ref, seq_used, batch_cnt, pre_release_callback);
  }

  if (immutable_db_options_.enable_pipelined_write) {//pipeline write
    return PipelinedWriteImpl(write_options, my_batch, callback, log_used,
                              log_ref, disable_memtable, seq_used);
  }
```
通过一个Writer结构封装一系列参数，声明一个stopwatch结构，然后把封装了数据的writer通过JoinBatchGroup添加到write_thread中，接下来根据情况进行判断。

```
		PERF_TIMER_GUARD(write_pre_and_post_process_time);
        //构建一个WriteThread::Writer
        WriteThread::Writer w(write_options, my_batch, callback, log_ref,
                              disable_memtable, batch_cnt, pre_release_callback);

        if (!write_options.disableWAL) {
            RecordTick(stats_, WRITE_WITH_WAL);
        }

        StopWatch write_sw(env_, immutable_db_options_.statistics.get(), DB_WRITE);
```

JoinBatchGroup函数内，WriteThread内部会尝试LinkOne判断该writer是不是第一个，如果是则将该Writer的状态设置成`STATE_GROUP_LEADER`，否则将会等待直到状态改变。

插播一下WriteThread的五种状态：
```
enum State : uint8_t {
	//writer的初始状态，等待JoinBatchGroup，后续可能变成
    //STATE_GROUP_LEADER、STATE_COMPLETE、STATE_PARALLEL_FOLLOWER
    STATE_INIT = 1,

    //通知一个writer现在变成了leader并且他需要创建一个write batch gorup 
    STATE_GROUP_LEADER = 2,

    //通知一个writer变成了一个memtable writer group的leader，leader要么将整个
    //group写到memtable，要么调用LaunchParallelMemtableWrite
    //发起一次并行写memtable 
    STATE_MEMTABLE_WRITER_LEADER = 4,

    // 告知一个writer变成了一个parallel memtable writer，writer应该将其batch同步
    //地应用到memtable并调用CompleteParallelMemtableWriter
    STATE_PARALLEL_MEMTABLE_WRITER = 8,

    //已经成功写入的writer
    STATE_COMPLETED = 16,

    // 告知thread需要等待StateMutex或者StateCondVar
    STATE_LOCKED_WAITING = 32, 
};
```

如果writer的state为WriteThread::STATE_PARALLEL_MEMTABLE_WRITER，说明当前的writer需要进行并行的memtable写

ShouldWriteToMemtable跟状态有关没有特别的判断逻辑，所以首先就先将batch写入memtbale
```
    // we are a non-leader in a parallel group

    if (w.ShouldWriteToMemtable()) {
      PERF_TIMER_STOP(write_pre_and_post_process_time);
      PERF_TIMER_GUARD(write_memtable_time);

      ColumnFamilyMemTablesImpl column_family_memtables(
          versions_->GetColumnFamilySet());
      w.status = WriteBatchInternal::InsertInto(
          &w, w.sequence, &column_family_memtables, &flush_scheduler_,
          write_options.ignore_missing_column_families, 0 /*log_number*/, this,
          true /*concurrent_memtable_writes*/, seq_per_batch_, w.batch_cnt);

      PERF_TIMER_START(write_pre_and_post_process_time);
    }
```
通过CompleteParallelMemtableWrite函数判断当前这个writer是不是writegroup里面最后一个完成的，如果不是则不用执行里面的操作，如果是则需要执行退出的操作，
退出操作就是执行回调函数之后并且更新sequence number

```
    if (write_thread_.CompleteParallelMemTableWriter(&w)) {
      // we're responsible for exit batch group
      for (auto* writer : *(w.write_group)) {
        if (!writer->CallbackFailed() && writer->pre_release_callback) {
          assert(writer->sequence != kMaxSequenceNumber);
          Status ws = writer->pre_release_callback->Callback(writer->sequence,
                                                             disable_memtable);
          if (!ws.ok()) {
            status = ws;
            break;
          }
        }
      }
      // TODO(myabandeh): propagate status to write_group
      auto last_sequence = w.write_group->last_sequence;
      versions_->SetLastSequence(last_sequence);
      MemTableInsertStatusCheck(w.status);
      write_thread_.ExitAsBatchGroupFollower(&w);
    }
    assert(w.state == WriteThread::STATE_COMPLETED);
    // STATE_COMPLETED conditional below handles exit

    status = w.FinalStatus();
```
如果writer的state为WriteThread::STATE_COMPLETED，则返回，至此完成了parallel memtable write
```
if (w.state == WriteThread::STATE_COMPLETED) {
            if (log_used != nullptr) {
                *log_used = w.log_used;
            }
            if (seq_used != nullptr) {
                *seq_used = w.sequence;
            }
            // write is complete and leader has updated sequence
            return w.FinalStatus();
        }
```
接下来是当前的writer成为leader的情况
```
assert(w.state == WriteThread::STATE_GROUP_LEADER);

        // Once reaches this point, the current writer "w" will try to do its write
        // job.  It may also pick up some of the remaining writers in the "writers_"
        // when it finds suitable, and finish them in the same write batch.
        // This is how a write job could be done by the other writer.
        WriteContext write_context;
        WriteThread::WriteGroup write_group;
        bool in_parallel_group = false;
        uint64_t last_sequence = kMaxSequenceNumber;
        if (!two_write_queues_) {
            last_sequence = versions_->LastSequence();
        }

        mutex_.Lock();

        bool need_log_sync = write_options.sync;
        bool need_log_dir_sync = need_log_sync && !log_dir_synced_;
```
首先进行预写(PreProcessWrite)，判断当前的memtable是不是写满了，然后进行刷写以及更换新的memtable
```
if (!two_write_queues_ || !disable_memtable) {
            //预写过程，判断memtbale是不是需要更换
            // With concurrent writes we do preprocess only in the write thread that
            // also does write to memtable to avoid sync issue on shared data structure
            // with the other thread
            //在并行写的时候，仅在写入memtbale的线程中进行preprocess
            //来避免与另一个线程的共享数据结构（memtable？）中的同步操作

            // PreprocessWrite does its own perf timing.
            PERF_TIMER_STOP(write_pre_and_post_process_time);

            status = PreprocessWrite(write_options, &need_log_sync, &write_context);

            PERF_TIMER_START(write_pre_and_post_process_time);
        }
log::Writer *log_writer = logs_.back().writer;

        mutex_.Unlock();

        // Add to log and apply to memtable.  We can release the lock
        // during this phase since &w is currently responsible for logging
        // and protects against concurrent loggers and concurrent writes
        // into memtables

        TEST_SYNC_POINT("DBImpl::WriteImpl:BeforeLeaderEnters");
```
然后对writer进行group操作，EnterAsBatchGroupLeader会在一定的大小阈值（1MB/1MB+128K）内将别的writer加入到当前leader的write_group中
```
last_batch_group_size_ =
                write_thread_.EnterAsBatchGroupLeader(&w, &write_group);
```

接着对计算整个group的size并判断是否能够并行写
```
if (status.ok()) {
            // Rules for when we can update the memtable concurrently
            // 1. supported by memtable
            // 2. Puts are not okay if inplace_update_support
            // 3. Merges are not okay
            //
            // Rules 1..2 are enforced by checking the options
            // during startup (CheckConcurrentWritesSupported), so if
            // options.allow_concurrent_memtable_write is true then they can be
            // assumed to be true.  Rule 3 is checked for each batch.  We could
            // relax rules 2 if we could prevent write batches from referring
            // more than once to a particular key.
            //是否允许并行写
            bool parallel = immutable_db_options_.allow_concurrent_memtable_write && write_group.size > 1;//允许并行memtable写以及write_group中有大于1个写请求
            size_t total_count = 0;
            size_t valid_batches = 0;
            size_t total_byte_size = 0;
            for (auto *writer : write_group) {
                if (writer->CheckCallback(this)) {
                    valid_batches += writer->batch_cnt;
                    if (writer->ShouldWriteToMemtable()) {
                        total_count += WriteBatchInternal::Count(writer->batch);
                        parallel = parallel && !writer->batch->HasMerge();//对应第3点
                    }

                    total_byte_size = WriteBatchInternal::AppendedByteSize(
                            total_byte_size, WriteBatchInternal::ByteSize(writer->batch));
                }
            }
            // Note about seq_per_batch_: either disableWAL is set for the entire write
            // group or not. In either case we inc seq for each write batch with no
            // failed callback. This means that there could be a batch with
            // disalbe_memtable in between; although we do not write this batch to
            // memtable it still consumes a seq. Otherwise, if !seq_per_batch_, we inc
            // the seq per valid written key to mem.
            size_t seq_inc = seq_per_batch_ ? valid_batches : total_count;

            const bool concurrent_update = two_write_queues_;
            // Update stats while we are an exclusive group leader, so we know
            // that nobody else can be writing to these particular stats.
            // We're optimistic, updating the stats before we successfully
            // commit.  That lets us release our leader status early.
            auto stats = default_cf_internal_stats_;
            stats->AddDBStats(InternalStats::NUMBER_KEYS_WRITTEN, total_count,
                              concurrent_update);
            RecordTick(stats_, NUMBER_KEYS_WRITTEN, total_count);
            stats->AddDBStats(InternalStats::BYTES_WRITTEN, total_byte_size,
                              concurrent_update);
            RecordTick(stats_, BYTES_WRITTEN, total_byte_size);
            stats->AddDBStats(InternalStats::WRITE_DONE_BY_SELF, 1, concurrent_update);
            RecordTick(stats_, WRITE_DONE_BY_SELF);
            auto write_done_by_other = write_group.size - 1;
            if (write_done_by_other > 0) {
                stats->AddDBStats(InternalStats::WRITE_DONE_BY_OTHER, write_done_by_other,
                                  concurrent_update);
                RecordTick(stats_, WRITE_DONE_BY_OTHER, write_done_by_other);
            }
            MeasureTime(stats_, BYTES_PER_WRITE, total_byte_size);

            if (write_options.disableWAL) {
                has_unpersisted_data_.store(true, std::memory_order_relaxed);
            }

            PERF_TIMER_STOP(write_pre_and_post_process_time);
```
接着写Write ahead log，这里分了两种情况，如果two_write_queues_为true表示将写memtable和其他写分开成两个queue（代码注释直译，具体实现还没找到），最终的区别是调用函数的区别，一个是WriteToWAL，另一个是ConcurrentWriteToWAL，最后更新sequence
```
            if (!two_write_queues_) {
                if (status.ok() && !write_options.disableWAL) {
                    PERF_TIMER_GUARD(write_wal_time);
                    //写到log中
                    status = WriteToWAL(write_group, log_writer, log_used, need_log_sync,
                                        need_log_dir_sync, last_sequence + 1);
                }
            } else {
                if (status.ok() && !write_options.disableWAL) {
                    PERF_TIMER_GUARD(write_wal_time);
                    // LastAllocatedSequence is increased inside WriteToWAL under
                    // wal_write_mutex_ to ensure ordered events in WAL
                    //并行写到log去
                    status = ConcurrentWriteToWAL(write_group, log_used, &last_sequence,
                                                  seq_inc);
                } else {
                    // Otherwise we inc seq number for memtable writes
                    last_sequence = versions_->FetchAddLastAllocatedSequence(seq_inc);
                }
            }
            assert(last_sequence != kMaxSequenceNumber);
            const SequenceNumber current_sequence = last_sequence + 1;
            last_sequence += seq_inc;
```

写memtable，分成两种情况，分别是非并行写以及并行写，非并行写的情况由当前线程完成所有写的内容，并行写的情况下所有的follower自己完成自己的写
```
if (status.ok()) {
                PERF_TIMER_GUARD(write_memtable_time);

                if (!parallel) {
                    // w.sequence will be set inside InsertInto
                    //非并行，由当前线程写所有的内容
                    w.status = WriteBatchInternal::InsertInto(
                            write_group, current_sequence, column_family_memtables_.get(),
                            &flush_scheduler_, write_options.ignore_missing_column_families,
                            0 /*recovery_log_number*/, this, parallel, seq_per_batch_);
                } else {
                    //并行，所有的follower自己写自己的内容
                    SequenceNumber next_sequence = current_sequence;
                    // Note: the logic for advancing seq here must be consistent with the
                    // logic in WriteBatchInternal::InsertInto(write_group...) as well as
                    // with WriteBatchInternal::InsertInto(write_batch...) that is called on
                    // the merged batch during recovery from the WAL.
                    for (auto *writer : write_group) {
                        if (writer->CallbackFailed()) {
                            continue;
                        }
                        writer->sequence = next_sequence;
                        if (seq_per_batch_) {
                            assert(writer->batch_cnt);
                            next_sequence += writer->batch_cnt;
                        } else if (writer->ShouldWriteToMemtable()) {
                            next_sequence += WriteBatchInternal::Count(writer->batch);
                        }
                    }
                    write_group.last_sequence = last_sequence;
                    write_group.running.store(static_cast<uint32_t>(write_group.size),
                                              std::memory_order_relaxed);
                    write_thread_.LaunchParallelMemTableWriters(&write_group);
                    in_parallel_group = true;

                    // Each parallel follower is doing each own writes. The leader should
                    // also do its own.
                    //所有的follower并行地开始写自己的writer
                    if (w.ShouldWriteToMemtable()) {
                        ColumnFamilyMemTablesImpl column_family_memtables(
                                versions_->GetColumnFamilySet());
                        assert(w.sequence == current_sequence);
                        w.status = WriteBatchInternal::InsertInto(
                                &w, w.sequence, &column_family_memtables, &flush_scheduler_,
                                write_options.ignore_missing_column_families, 0 /*log_number*/,
                                this, true /*concurrent_memtable_writes*/, seq_per_batch_,
                                w.batch_cnt);
                    }
                }
                if (seq_used != nullptr) {
                    *seq_used = w.sequence;
                }
            }
        }
        PERF_TIMER_START(write_pre_and_post_process_time);
```
后续更新，比如更新log内容 以及执行完成的回调工作
```
if (!w.CallbackFailed()) {
            WriteStatusCheck(status);
        }

        if (need_log_sync) {
            //同步log
            mutex_.Lock();
            MarkLogsSynced(logfile_number_, need_log_dir_sync, status);
            mutex_.Unlock();
            // Requesting sync with two_write_queues_ is expected to be very rare. We
            // hence provide a simple implementation that is not necessarily efficient.
            if (two_write_queues_) {
                if (manual_wal_flush_) {
                    status = FlushWAL(true);
                } else {
                    status = SyncWAL();
                }
            }
        }

        bool should_exit_batch_group = true;
        if (in_parallel_group) {
            // CompleteParallelWorker returns true if this thread should
            // handle exit, false means somebody else did
            //如果是一个并行写的group，就需要判断当前这个writer是不是需要处理退出流程（也就是最后一个完成的）
            should_exit_batch_group = write_thread_.CompleteParallelMemTableWriter(&w);
        }
        if (should_exit_batch_group) {
            //当前的writer应该执行退出流程
            if (status.ok()) {
                for (auto *writer : write_group) {
                    if (!writer->CallbackFailed() && writer->pre_release_callback) {
                        assert(writer->sequence != kMaxSequenceNumber);
                        Status ws = writer->pre_release_callback->Callback(writer->sequence,
                                                                           disable_memtable);
                        if (!ws.ok()) {
                            status = ws;
                            break;
                        }
                    }
                }
                versions_->SetLastSequence(last_sequence);
            }
            MemTableInsertStatusCheck(w.status);
            write_thread_.ExitAsBatchGroupLeader(write_group, status);
        }

        if (status.ok()) {
            status = w.FinalStatus();
        }
        return status;
```

