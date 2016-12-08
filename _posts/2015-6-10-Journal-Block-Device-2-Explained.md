---
layout: post
title: Journal Block Device 2 Explained
---

# Contents

1. [Introduction][section-introduction]
2. [Start Transaction][section-start-transaction]
2.1. [Start New Handle][section-start-handle]
2.1.1. [Build Handle for Task][section-build-handle]
2.1.2. [Build Transaction and Connect it to Journal State]
                                                    [section-build-transaction]
2.1.3. [Connect the Transaction to the Handle][section-connect-transaction]
2.2. [Nested Transaction of the Same Task][section-nested-transaction]
2.3. [Nested Transaction of Different Tasks][section-nested-transac-tasks]
3. [Get Write Access for Buffer Head][section-get-write-access]
3.1. [Build Journal Head and Connect it to Transaction]
                                                [section-build-connect-jhead]
3.1.1. [Build Journal Head][section-build-jhead]
3.1.2. [Connect Journal Head to Buffer Head][section-connect-jhead]
3.1.3. [Connect Journal Head to Transaction][section-connect-jhead-transaction]
3.2. [Frozen Data][section-frozen-data]
4. [Mark Buffer as Containing Dirty Metadata][section-mark-bhead-dirty]
5. [Close Transaction and Handle][section-close-handle]
6. [Commit Transaction][section-commit-transaction]
6.1. [Transaction State Transition][section-tran-state-trans]
6.2. [Commit Layout][section-commit-layout]
7. [Checkpoint Transactions][section-checkpoint-trasaction]
8. [Revocation][section-revoke]
9. [Escaping][section-escape]

---

# 1. Introduction [section-introduction]

---

Journal Block Device

EXT4 에서 journal 파일 자체는 정해진 inode 넘버로 설정되며 해당 저널 inode 의
저널모드는 `EXT4_INODE_WRITEBACK_DATA_MODE` 이 되며 `inode->i_flags` 에서
`EXT4_STATE_ORDERED_MODE` 해제된다. 결과적으로 저널 inode 의 `aops` 는
`ext4_aops` 로 설정된다.

# 2. Start Transaction [section-start-transaction]

---

## 2.1. Start New Handle [section-start-handle]

    jbd2__journal_start(journal, nblock, ...)
        handle = new_handle(nblock)                         ... 2.1.1.
        start_this_handle(journal, handle)                  ... 2.1.2.
            new_transaction = kmem_cache_zalloc()
            jbd2_get_transaction(journal, new_transaction)
            add_transaction_credits(journal, blocks)
        connect_transaction_to_handle                       ... 2.1.3.

### 2.1.1. Build Handle for Task [section-build-handle]

    * new_handle()

    +- handle_t (new) --+
    | h_buffer_credits  | (= nblock)
    | h_ref             | (= 1)
    +-------------------+

`h_buffer_credits` 멤버변수는 `jbd2__journal_start(nblock)` 에 요청된 `nblock`
만큼 할당된다.
`handle_t` 는 `current->journal_info` 에 할당되며 파일시스템관련 커널작업 진행
중인 태스크 당 하나씩 할당된다. 다른 말로 하면 파일시스템관련 커널작업이 진행
중인 프로세스마다 `handle` 이 새로 할당된다. `journal_info` 에 `handle` 이 이미
할당되어있다면 `jbd2__journal_start()` 콜은 `h_ref` 를 증가시키는 것만으로
완료된다.

### 2.1.2. Build Transaction and Connect it to Journal State [section-build-transaction]

    * start_this_handle()
          jbd2_get_transaction()
          add_transaction_credits()

    +- journal_t -----------+---------------+
    | j_running_transaction | <-------------|---+
    | j_transaction_squence | -- (= ++) ----|---|---+
    | j_commit_interval     | --------------|---|---|---+
    +-----------------------+               |   |   |   |
                                            |   |   |   |
    +- transaction_t (new) -+ --------------|---+   |   |
    | t_journal             | <-------------+       |   |
    | t_state               | (= T_RUNNING)         |   |
    | t_tid                 | <---------------------+   |
    | t_expires             | <-- (= jiffies +) --------+
    | t_handle_count        | (= 0)
    | t_updates             | (= 0)
    | t_outstanding_credits | (= blocks)
    | t_start               | (= jiffies)
    | t_requested           | (= 0)
    +-----------------------+

`transaction_t` 은 `journal->j_running_transaction == NULL` 인 경우에만 새로
할당되며 이는 `journal->j_state_lock` 을 대상으로 `write_lock` 으로 보호되어
하나의 인스턴스만 유지되도록 보장된다.
`t_tid` 는 `journal_t->j_transaction_squence` (매 트렌젝션 생성마다 증가하는)
으로 초기화된다.
`t_outstanding_credits` 에는 해당 트랜젝션에 포함될 수 있는 변경가능한 블럭수가
저장된다.
`t_expires` 에는 `journal_t->j_commit_interval + jiffies` 값이 저장되어 해당
트랜젝션이 일정시간 이상 커밋되지 않는 상황을 피하도록 한다.

>`journal_t` 는 파일시스템마다 하나씩 할당된다.

### 2.1.3. Connect the Transaction to the Handle [section-connect-transaction]

    * start_this_handle(journal, handle)

    +- journal_t -----------+
    | j_running_transaction | <---------+
    +-----------------------+           |
                                        |
    +- task A ------+                   |
    | journal_info  | <---------+       |
    +---------------+           |       |
                                |       |
    +- handle_t ------------+ --+       |
    | h_transaction         | <---------+
    | h_requested_credits   | (= blocks)|
    +-----------------------+           |
                                        |
    +- transaction_t -------+ ----------+
    | t_state               | (== T_RUNNING)
    | t_updates             | (= ++)
    | t_handle_count        | (= ++)
    | t_outstanding_credits | (+= blocks)
    +-----------------------+

`t_updates` 는 `jdb2__journal_start()` 호출마다 그 수가 1 씩 더해지게 되며
`jbd2_journal_stop()` 에서 그 수가 줄어든다. 이 멤버변수는
`jbd2_journal_commit_transaction()` 에서 `j_running_transaction` 트랜젝션의
커밋 직전에 이 수가 `0` 이 될 때까지 대기하도록 함으로써 진행 중인 트랜젝션이
없는 경우에만 커밋이 진행되도록 한다.
`t_outstanding_credits` 는 저널에 관련된 버퍼들의 수를 나타내며
`jbd2_journal_stop()` 에서 `handle_t->h_buffer_credits` 값 만큼 감소된다.

---

## 2.2. Nested Transaction of the Same Task [section-nested-transaction]

    * jbd2__journal_start(journal, nblocks, ...)
          jbd2__journal_start(journal, nblocks, ...)

    +- current -----+
    | journal_info  | <---------+
    +---------------+           |
                                |
    +- handle_t --------+ ------+
    | h_ref             | (= ++)
    +-------------------+

하나의 테스크가 복수로 `jbd2__start_journal()` 을 호출하는 경우
`current->journal_info != NULL` 이기 때문에 `handle_t` 의 레퍼런스카운터의
업데이트만으로 종료된다.

### 2.3. Nested Transaction of Different Tasks [section-nested-transac-tasks]

    * start_this_handle(journal, handle)

    +- journal_t -----------+
    | j_running_transaction | <---------+
    +-----------------------+           |
                                        |
    +- task A ------+                   |   +- task B ------+
    | journal_info  | <---------+       |   | journal_info  | <---------+
    +---------------+           |       |   +---------------+           |
                                |       |                               |
    +- handle_t ------------+ --+       |   +- handle_t ------------+ --+
    | h_transaction         | <---------+   | h_transaction         | <-+
    | h_requested_credits   | (= blocks)|   | h_requested_credits   |   |
    +-----------------------+           |   +-----------------------+   |
                                        |                               |
    +- transaction_t -------+ ----------+-------------------------------+
    | t_state               | (== T_RUNNING)
    | t_updates             | (= ++)
    | t_handle_count        | (= ++)
    | t_outstanding_credits | (+= blocks)
    +-----------------------+

복수의 태스크가 커널제어 흐름에서 `jbd2__start_journal()` 을 호출하게 되면
`add_transaction_credits()` 함수에서 `transaction_t` 의 각 멤버변수가 업데이트
되는 것으로 완료된다. `transaction_t` 는 락에 의해 하나로 유지된다.

# 3. Get Write Access for Buffer Head [section-get-write-access]

---

    jbd2_journal_get_write_access(handle_t, buffer_head)
        journal_head = jbd2_journal_add_journal_head(buffer_head)
        do_get_write_access(handle_t, journal_head)
            if (buffer_dirty(buffer_head))
                clear_buffer_dirty(buffer_head)
                set_buffer_jbddirty(buffer_head)
            __jbd2_journal_file_buffer(journal_head, transaction, BJ_Reserved)
                if (test_clear_buffer_dirty(buffer_head) ||
                    test_clear_buffer_jbddirty(buffer_head))
                        was_dirty = 1;
                if (journal_head->b_transaction)
                    __jbd2_journal_temp_unlink_buffer(journal_head)
                        if (test_clear_buffer_jbddirty(bh))
                            mark_buffer_dirty(bh)
                if (was_dirty)
                    set_buffer_jbddirty(buffer_head)

## 3.1. Build Journal Head and Connect it to Transaction [section-build-connect-jhead]
### 3.1.1. Build Journal Head [section-build-jhead]

    * jbd2_journal_get_write_access(buffer_head)
          if (!buffer_jbd(buffer_head))
              new_jh = journal_alloc_journal_head()

    +- journal_head (new) --+
    | b_bh                  | (= 0)
    | b_jcount              | (= 0)
    | b_transaction         | (= 0)
    | b_modified            | (= 0)
    +-----------------------+

`journal_head` 는 저널링을 사용하려는 `buffer_head` 에 대하여 하나씩 할당된다.
이미 `buffer_head` 에 `journal_head` 가 할당되어있으면 `b_count` 의 증가만으로
완료된다.

### 3.1.2. Connect Journal Head to Buffer Head [section-connect-jhead]

    * jbd2_journal_get_write_access(handle_t, buffer_head)
          jbd2_journal_add_journal_head(buffer_head)

    +- buffer_head -+ --------------+
    | b_state       | (= BH_JBD)    |
    | b_private     | <-----+       |
    +---------------+       |       |
                            |       |
    +- journal_head ----+ --+       |
    | b_bh              | <---------+
    | b_jcount          | <- (= ++)
    | b_transaction     | (= 0)
    | b_modified        | (= 0)
    +-------------------+

`buffer_head` 에 `journal_head` 하나를 할당한 후에는 `b_state` 에 `BH_JBD`
비트를 설정한 후, `journal_head->b_bh` 에 `buffer_head` 를 할당하고
`buffer_head->b_private` 에 `journal_head` 인스턴스를 할당함으로써 둘을
상호연결한다.

### 3.1.3. Connect Journal Head to Transaction [section-connect-jhead-transaction]

    * jbd2_journal_get_write_access(handle_t, buffer_head)
          jbd2_journal_add_journal_head(buffer_head)
          do_get_write_access(handle_t, journal_head)
            if (buffer_dirty(buffer_head))
                clear_buffer_dirty(buffer_head)
                set_buffer_jbddirty(buffer_head)
            if (!journal_head->b_transaction)
                __jbd2_journal_file_buffer(jh, transaction_t, BJ_Reserved)

    +- handle_t ----+
    | h_transaction | --------------+
    +---------------+               |
                                    |
    +- transaction -----+ <---------+
    | t_reserved_list   | <---------|---+
    +-------------------+           |   |
                                    |   |
    +- journal_head ----+ ------+---|---+
    | b_tnext           | <-----+   |
    | b_tprev           | <-----+   |
    | b_jcount          | (= ++)    |
    | b_transaction     | <---------+
    | b_modified        | (= 0)
    | b_bh              | <-----+
    +-------------------+       |
                                |
    +- buffer_head -+ ----------+
    |               |
    +---------------+

이 시점에 `journal_head->b_transaction` 을 현재 활성화된 트랜젝션인
`handle_t->h_transaction` 으로 초기화하고 이 트랜젝션의 임시 리스트라고 할 수
있는 `transaction_t`->t_reserved_list` 에 연결한다.

> **NOTE**
> 먼저 `BH_Dirty` 인지 확인하여 해당 플래그를 해제하는데 이미 버퍼가 dirty 인
> 경우는 해당 버퍼가 이미 커밋완료되어 다시 dirty 가 된 경우로 `buffer_head`
> 에서 `BH_Dirty` 플래그를 해제하여 mm/fs 레이어에서 해당 버퍼가 다시 clean
> 으로 보이게 하여 I/O 요청을 하지 못하게 하려는 목적이 있다.
> 하지만 아직 page 는 `PG_dirty` 이고 페이지 캐쉬트리에도 `PAGECACHE_TAG_DIRTY`
> 이 설정되어 있기 때문에 writeback 코드는 해당 page 를 일반적 인 흐름에서
> `PG_dirty` 해제 후, `PG_writeback` 플래그 및 `PAGECACHE_TAG_WRITEBACK` 태그를
> 설정하고 쓰기를 시도하려고 한다. 하지만 본 함수에서 `buffer_head` 를 clean 한
> 상태이므로 만일 page 에 속한 모든 `buffer_head` 가 clean 하다면 I/O 요청은
> 이루어 지지 않고 그 시점에 완료된 것으로 간주하여 `PG_writeback` 플래그 및
> `PAGECACHE_TAG_WRITEBACK` 태그를 해제한다.

#### In Case of Multiple Journal Heads in a Transaction [section-multi-jhead]

    +- transaction -----+ ------+-------------------------------+--
    | t_reserved_list   | <-----|---+                           |
    +-------------------+       |   |                           |
                                |   |                           |
    +- journal_head ----+ ------|---+   +- journal_head ----+   |
    | b_transaction     | <-----+       | b_transaction     | <-+
    |                   |               |                   |
    | b_tnext           | <-----------> | b_tprev           | <----> ...
    | b_tprev           |               | b_tnext           |
    +-------------------+               +-------------------+

`_t_reserved_list` 는 `struct journal_head *` 형이며 `journal_head` 들은 자신의
멤버변수인 `b_tnext`, `b_tprev` 를 이용해 연결된다.

## 3.2. Frozen Data [section-frozen-data]

    jbd2_journal_get_write_access(handle_t, buffer_head)
        do_get_write_access(handle_t, journal_head)
            transaction = handle->h_transaction

            if (jh->b_frozen_data)
                jh->b_next_transaction = transaction
                goto done

먼저 `b_frozen_data` 가 할당되어있는지 확인하는데 이 경우는 이미
jbd2_journal_get_write_access()` 가 여러번 호출되면서 아래의 경우에 해당되어
복사공간 할당 및 복사가 끝난 상태로 이러한 경우에는 `b_next_transaction` 에
현재 transaction 을 할당해놓음으로써 커밋처리 중 `t_forget` 리스트를 처리할 때
다시 트랜젝션의 대상으로 본 `buffer_head` 를 다시 추가하기 위함이다.

            if (journal_head->b_transaction
                && journal_head->b_transaction != transaction)

`if (jh->b_transaction && jh->b_transaction != transaction)`
do_get_write_access() 호출시점에 `jh->b_transaction != NULL` 이 의미하는 것은
해당 `buffer_head` 에 대해 이미
`__jbd2_journal_file_buffer(journal_head, transaction_t, jlist)` 이 호출되어
`transaction_t` 의 리스트 중 하나에 등록되었다는 의미이다.
(`BJ_Metadata` or `BJ_Forget` or `BJ_Shadow` or `BJ_Reserved`)
`jh->b_transaction != transaction` 이 의미하는 것은 해당 `journal_head` 이
`journal_t->j_running_transaction` 이 아닌 `journal_->j_commining_transaction`
 이나 `journal_t->j_checkpoint_transactions` 트랜젝션에 연결된 경우이다.
결론적으로 다른 트랜젝션의 임의의 리스트에 연결되었다는 의미이다.

                if (buffer_shadow(buffer_head))
                    wait_on_bit_io(&buffer_head->b_state,
                                   BH_Shadow, TASK_UNINTERRUPTIBLE)
                    goto repeat

해당 파일 시스템용 `buffer_head` 가 커밋처리과정에서 이 `head_head` 에 대해
저널쓰기용 `buffer_head` 를 새롭게 할당하고 파일시스템용 `buffer_head` 는
`BJ_Shadow` 리스트에 연결한 경우이다. 저널쓰기용 `buffer_head` 가 완료되면
`journal_end_buffer_io_sync()` 에 의해 파일시스템용 `buffer_head` 의
`BH_Shadow` 를 해제하고 본 프로세스를 깨운 후 처음부터 다시 시작한다.

>`jbd2_journal_write_metadata_buffer(transactiont, *jh_in, **bh_out, blocknr)`
>참고

                if (jh->b_jlist == BJ_Metadata)
                    if (!frozen_buffer)
                        bd_unlock_bh_state(bh)
                        frozen_buffer = jbd2_alloc(jh2bh(jh)->b_size)
                        goto repeat
                    jh->b_frozen_data = frozen_buffer
                    frozen_buffer = NULL
                    need_copy = 1
                jh->b_next_transaction = transaction

`journal_head` 가 속한 트랜젝션이 `T_COMMIT` 상태이거나 그 이전 상태임을
나타내며 그러한 상태에서는 `buffer_head->b_page` 의 내용이 곧 쓰기에 들어갈
예정이므로 b_frozen_data 를 할당할 준비를 한다.
`b_next_transaction` 에 트랜젝션이 할당되면 커밋처리중 `t_forget` 리스트를
처리하는 도중에 `_jbd2_journal_refile_buffer()` 에서 관련된 리스트에 다시
`journal_head` 를 추가한다.

> jbd2_journal_write_metadata_buffer() 참고

# 4. Mark Buffer as Containing Dirty Metadata [section-mark-bhead-dirty]

---

    jbd2_journal_dirty_metadata(handle_t, buffer_head)
        if (jh->b_modified == 0)
            jh->b_modified = 1
            handle->h_buffer_credits--
        set_buffer_jbddirty(buffer_head)
        __jbd2_journal_file_buffer(journal_head, transaction_t, BJ_Metadata)

    +- handle_t --------+
    | h_transaction     | ----------+
    | h_buffer_credits  | (= --)    |
    +-------------------+           |
                                    |
    +- transaction_t ---+ <---------+-------------------------------+
    | t_nr_buffers      | (= ++)    |                               |
    | t_buffers         | <---------|---+                           |
    +-------------------+           |   |                           |
                                    |   |                           |
    +- journal_head ----+ ----------|---+   +- journal_head ----+   |   +- ----
    | b_tnext           | <---------|---|-> | b_tnext           | <-|-> |
    | b_tprev           |           |   |   | b_tprev           |   |   |
    | b_jcount          | <- (= ++) |   |   | b_jcount          |   |   | ...
    | b_transaction     | <---------+   |   | b_transaction     | <-+   |
    | b_modified        | (= 1)         |   | b_modified        |       |
    | b_bh              | <-----+       |   | b_bh              | <-+   |
    +-------------------+       |       |   +-------------------+   |   +------
                                |       |                       |   |
    +- buffer_head -+ ----------+       |   +- buffer_head -+ <-|---+   +- ----
    | b_private     | <-----------------+   | b_private     | <-+       | ...
    | b_state       | (= BH_JBD|BH_JBDDirty)| b_state       |           |
    +---------------+                       +---------------+           +------

`journal_head->b_modified` 가 0 이므로 `handle_t->h_buffer_credits` 를
감소시킨다.
이후 `buffer_head` 에 `BH_JBDDirty` 을 설정하여 저널영역에 써야할 데이터가
있음을 나타낸다.
마지막으로 `BJ_Metadata` 는 `transaction_t->t_buffers` 에 `journal_head` 를
연결한다.

# 5. Close Transaction and Handle [section-close-handle]

---

    jbd2_journal_stop(handle_t)

    +- current -----+
    | journal_info  | (= NULL) x----+
    +---------------+               |
                                    |
    +- handle_t --------+ x---------+
    | h_ref             | (== 0)
    | h_buffer_credits  | --------------+
    | h_transaction     | <-----+       |
    +-------------------+       |       |
                                |       |
    +- transaction_t -------+ --+       |
    | t_outstanding_credits | <- (-=) --+
    +-----------------------+

종료시점에 `handle_t` 에 처리한 블럭 수인 `h_buffer_credits` 를
`transaction_t->t_outstanding_credits` 에서 빼서 블럭수를 보정한다.
`current->journal_info` 를 `NULL` 로 만들어서 진행중인 저널조작이 없을음
표시한다.

# 6. Commit Transaction [section-commit-transaction]

---

## Brief Sequence

    ---> data blocks sent to orignal FS (ordered)

    ---> start block plug

    ---> revoke records sent to JOURNAL
    ---> descriptors & tag blocks sent to JOURNAL

    <=== data blocks written on original FS (ordered)
    if (commit_transaction->t_need_data_flush &&
        (journal->j_fs_dev != journal->j_dev) &&
        (journal->j_flags & JBD2_BARRIER))
        <=== FS device flushed, which means all data block written (ordered)

    if (JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT)
        ---> commit log sent to JOURNAL

    <--- finish block plug

    <=== tag blocks written on JOURNAL
    <=== revoke & descriptors written on JOURNAL

    if (!JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT)
        ---> commit log sent to JOURNAL

    <=== commit log written on JOURNAL
    if (JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT && JBD2_BARRIER)
        <=== JOURNAL device flushed, which means all journal block written

    ---> mark dirty blocks written on JOURNAL again

## 6.1. Transaction State Transition [section-tran-state-trans]

### 6.1.1. T_RUNNING Transaction State

jbd2__journal_start() 함수에서 현재 journal->j_running_transaction
이 비어있는 경우 새로 할당되는 트랜젝션의 상태가 T_RUNNING 상태로
시작되며 journal->j_running_transaction 에 해당 트랜젝션이
할당된다.
이 시점에 자유롭게 저널요청이 수락되어 j_running_transaction 에
설정된 트랜젝션으로 포함된다.

### 6.1.2. T_LOCKED Transaction State

해당 트랜젝션이 잠겼음을 나타내며 이 상태를 벗어날 때까지
jbd2__journal_start() 함수가 대기한다.
또한 jbd2__journal_start() 로 시작된 저널요청들이
jbd2_journal_stop() 을 호출하여 모두 끝날 때까지 이 상태에서
대기한다.

### 6.1.3. T_FLUSH Transaction State

    +- journal_t ---------------+
    | j_running_transaction     | (= NULL)
    | j_committing_transaction  | <- (= j_running_transaction) -+
    +---------------------------+                               |
                                                                |
    +- transaction_t ---+---------------------------------------+
    | t_inode_list      | <-+-----------------------+
    +-------------------+   |                       |
                            |                       |
    +- jbd2_inode --+       |   +- jbd2_inode --+   |
    | i_list        | ------+   | i_list        | --+   ... ... ...
    | i_vfs_inode   | <-----+   | i_vfs_inode   |
    +---------------+       |   +---------------+
                            |
    +- address_space ---+---+
    | tadix_tree_root   |       ... ... ...
    +-------------------+

먼저 본 트랜젝션을 j_committing_transaction 으로 옮기고 j_running_transaction
을 NULL 로 만들어 새로운 트랜젝션이 생성될 수 있도록 한다.
j_wait_transaction_locked 에서 대기중인 프로세스를 깨워 새로운 트랜젝션을
시작하려는 프로세스를 깨운다. ordered 모드의 경우, 메타데이터 저널 요청과
관련된 데이터블럭 변경이 있다면 해당 데이터블럭을 포함하는 inode 들의 페이지들
을 쓰기 요청한다. (`ext4_write_end()` 에서 그러한 inode 들이 `t_inode_list` 에
연결된다.) data 모드의 경우에 이 리스트는 비어있다.

### 6.1.4. T_COMMIT Transaction State

이 트랜젝션의 t_buffers 에 연결된 저널 요청들 (journal_head) 을 훑으면서 해당
요청들을 저장할 저널영역의 블럭을 얻어내며 원래 블럭(original target block)의
저널 요청을 BJ_Shadow 리스트 (t_shadow_list)로 옮긴다.
커밋디스크립터를 생성하고 (journal_header_t + journal_block_tag_t) 얻어낸
저널영역 블럭에 저널데이터와 함께 쓰기 요청한다. (submit_bh(WRITE_SYNC), bh)
그 후 T_FLUSH 상태에서 쓰기 요청한 저널요청과 관련된 데이터 페이지들의 완료를
대기한다. (filemap_fdatawait(jinode->i_vfs_inode->i_mapping))

#### Before T_COMMIT State

    +- journal_t ---------------+
    | j_committing_transaction  | <-+
    +---------------------------+   |
                                    |
    +- transaction_t ---+ ----------+
    | t_buffers         | <---------|---+
    +-------------------+           |   |
                                    |   |
    +- journal_head ----+ ----------|---+   +- journal_head ----+       ...
    | b_transaction     | <---------+   |   | b_transaction     |
    | b_tprev/tnext     | <-------------|-> | b_tnext           | <-------
    | b_modified        | (1)           |   | b_modified        |
    | b_bh              | <-----+       |   | b_bh              | <-+
    +-------------------+       |       |   +-------------------+   |
                                |       |                           |
    +- buffer_head -+ ----------+       |   +- buffer_head -+-------+   ...
    | b_private     | <-----------------+   | b_private     |
    | b_jlist       | (BJ_Metadata)         | b_jlist       | (BJ_Metadata)
    | b_state       | (BH_JBD | BH_JBDDirty)| b_state       | (BH_JBD | ...)
    | b_page        | <-+                   | b_page        | <-+
    +---------------+   |                   +---------------+   |
                        |                                       |
    +- page ----+-------+                   +- page ----+-------+       ...
    |           |                           |           |
    +-----------+                           +-----------+

커밋 대상이 되는 `journal_head` 들이 `transaction_t->t_buffers` 에 연결되어
있으며 각 `journal_head` 에 원래 `buffer_head` 들이 clean 한 상태로 연결되어
있다. `page` 들 또한 clean 한 상태이다. mm/fs 레이어에서는 해당 `buffer_head`
들과 `page` 들이 clean 한 상태로 일반적인 writeback 루틴의 대상이 아니다.

#### Before Submitting Journal Buffers

    +- journal_t ---------------+
    | j_committing_transaction  | <-+
    +---------------------------+   |
                                    |
    +- transaction_t ---+ ----------+
    | t_buffers         | <---------|---+
    +-------------------+           |   |
                                    |   |
    +- journal_head ----+ ----------|---+   +- journal_head ----+       ...
    | b_transaction     | <---------+   |   | b_transaction     |
    | b_tprev/tnext     | <-------------|-> | b_tprev/next      | <-------
    | b_modified        | (1)           |   | b_modified        |
    | b_bh              | <-----+       |   | b_bh              |
    +-------------------+       |       |   +-------------------+
                                |       |
    +- buffer_head -+ ----------+       |   ...                         ...
    | b_private     | <-----------------+
    | b_jlist       | (BJ_Metadata)
    | b_state       | (= BH_JBD | BH_JBDDirty | BH_JWrite)
    | b_blocknr     | (original target block)
    | b_page        | <-+
    +---------------+   |
                        |
    +- page ----+-------+---+               ...                         ...
    |           |           |
    +-----------+           |
                            |
    +- buffer_head -----+   |               ...                         ...
    | b_page            | <-+
    | b_state           | (= BH_Dirty | BH_Mapped)
    | b_blocknr         | (journal block)
    | b_assoc_buffers   | ------+
    +-------------------+       |
                                |
    +- io_bufs (list_head) -+   |
    |                       | <-+
    +-----------------------+

I/O 를 요청하기에 앞서 `t_buffers` 에 연결된 `buffer_head->b_state` 에
`BH_JWrite` 플래그를 추가한다. (그렇다고 해서 이 `buffer_head` 에 대해 I/O 요청
을 보내는 것은 아니다.)
또한 저널영역에 `page` 의 데이터를 쓰기 위하여 저널 영역의 블럭을 가리키는
새로운 `buffer_head` 들을 생성하고 `BH_Dirty` 상태로 만들며 `page` 들을 그 곳에
연결한 후, I/O 완료를 대기할 수 있도록 임시 리스트인 `io_bufs` 리스트에 연결
해둔다.

> 생성한 `buffer_head` 에 기존 페이지를 연결하는 데신 새로운 페이지를 할당하고
> 기존 내용을 복사해 놓는 경우도 있다. ([Frozen Data](#section-frozen-data))

#### After Submitting Journal Buffers

    +- journal_t ---------------+
    | j_committing_transaction  | <-+
    +---------------------------+   |
                                    |
    +- transaction_t ---+ ----------+
    | t_buffers         | xxx       |
    | t_shadow_list     | <---------|---+
    +-------------------+           |   |
                                    |   |
    +- journal_head ----+ ----------|---+   +- journal_head ----+       ...
    | b_transaction     | <---------+   |   | b_transaction     |
    | b_tprev/tnext     | <-------------|-> | b_tprev/next      | <-------
    | b_modified        | (1)           |   | b_modified        |
    | b_bh              | <-----+       |   | b_bh              |
    +-------------------+       |       |   +-------------------+
                                |       |
    +- buffer_head -+ ----------+       |   ...                         ...
    | b_private     | <-----------------+
    | b_jlist       | (= BJ_Shadow)
    | b_state       | (= BH_JBD | BH_JBDDirty | BH_JWrite | BH_Shadow)
    | b_blocknr     | (original target block)
    | b_page        | <-+
    +---------------+   |
                        |
    +- page ----+-------+---+               ...                         ...
    | DATA      |           |
    +-----------+           |
                            |
    +- buffer_head -----+   |               ...                         ...
    | b_page            | <-+
    | b_state           | (= BH_Uptodate)
    | b_blocknr         | (journal block)
    | b_assoc_buffers   | ------+
    +-------------------+       |
                                |
    +- io_bufs (list_head) -+   |
    |                       | <-+
    +-----------------------+

원래 타겟 블럭을 가리키는 `buffer_head` 들을 `t_shadow_list` 로 옮긴다.
저널영역의 블럭을 가리기고 있는 `buffer_head` 에 대하여 I/O 요청을 보내기에
앞서 `BH_Dirty` 플래그를 제거하고 `submit_bh(WRITE_SYNC, bh)` 를 이용하여
I/O 요청을 보낸다.
`T_FLUSH` 상태에서 쓰기 요청을 보낸 `inode` 들의 페이지들의 완료를 대기한다.
이로써 원래 타겟 블럭에 쓰여져야할 데이터블럭은 쓰기가 완료되었다.

### 6.1.5. T_COMMIT_DFLUSH Transaction State

    +- journal_t ---------------+
    | j_committing_transaction  | <-+
    +---------------------------+   |
                                    |
    +- transaction_t ---+ ----------+
    | t_shadow_list     | xxx       |
    | t_forget          | <---------|---+
    +-------------------+           |   |
                                    |   |
    +- journal_head ----+ ----------|---+   +- journal_head ----+       ...
    | b_transaction     | <---------+   |   | b_transaction     |
    | b_tprev/tnext     | <-------------|-> | b_tprev/next      | <-------
    | b_modified        | (1)           |   | b_modified        |
    | b_bh              | <-----+       |   | b_bh              |
    +-------------------+       |       |   +-------------------+
                                |       |
    +- buffer_head -+ ----------+       |   ...                         ...
    | b_private     | <-----------------+
    | b_jlist       | (= BJ_Forget)
    | b_state       | (= BH_JBD | BH_JBDDirty)
    | b_blocknr     | (original target block)
    | b_page        | <-+
    +---------------+   |
                        |
    +- page ----+-------+                   ...                         ...
    | DATA      |
    +-----------+

JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT 가 활성화되어 있는 경우에는
journal_submit_commit_record() 을 이용하여 커밋레코드를 쓰기 요청한다. 비동기적
으로 먼저 커밋레코드를 쓰기요청하기 때문에 실제적으로 디스크에 저널데이터나
디스크립트보다 먼저 쓰여질 가능성이 있으며 복구시 이러한 경우를 검출하기 위해
비동기 커밋레코드를 활성화한 경우 자동적으로 사용되는
`JBD2_FEATURE_COMPAT_CHECKSUM` 기능을 이용하여 해결한다.

`T_COMMIT` 단계에서 임시 io_bufs 리스트에 연결해둔 I/O 요청한 `buffer_head` 의
완료를 대기한다. 대기완료된 `buffer_head` 에 대응되는 원래 타겟 블럭을 가리키는
`buffer_head` 들에서 `BH_JWrite` 플래그를 제거하고 `BJ_Forget` 리스트로 옮긴다.
`BH_Shadow` 는 I/O 완료콜백에서 제거된다.
또한 마찬가지로 `T_COMMIT` 단계에서 임시 `log_bufs` 리스트에 연결해둔 I/O 요청
한 디스크립터 버퍼의 완료도 대기한다. 쓰기 완료된 저널쓰기 용 `buffer_head` 는
해제한다.

### 6.1.6. T_COMMIT_JFLUSH Transaction State

    +- transaction_t ---+---------------+
    | t_checkpoint_list | <---------+   |
    +-------------------+           |   |
                                    |   |
    +- journal_head ----+-----------+   |       ...             ...
    | b_cp_transaction  | <---------|---+
    | b_cpprev/cpnext   | <---------|--------->
    | b_transaction     | (= NULL)  |
    | b_next_transaction|           |
    | b_bh              | <-+       |
    +-------------------+   |       |
                            |       |
    +- buffer_head -+-------+       |           ...             ...
    | b_private     | <-------------+
    | b_jlist       | (= BJ_None)
    | b_state       | (= BH_Dirty)
    | b_blocknr     | (original target block)
    | b_page        | <-+
    +---------------+   |
                        |
    +- page ----+-------+                       ...             ...
    | flags     | (= PG_dirty)
    | mapping   | <---------+
    +-----------+           |
                            |
    +- address_space ---+---+                   ...             ...
    | page_tree         | (= PAGECACHE_TAG_DIRTY)
    | host              | <-+
    +-------------------+   |
                            |
    +- inode ---+-----------+                   ...             ...
    | i_state   | (= I_DIRTY_PAGES)
    | i_wb_list | --------------+
    +-----------+               |
                                |
    +- backing_dev_info ----+   |               ...             ...
    | wb.b_dirty            | <-+
    +-----------------------+

`JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT` 이 비활성화되어있어 (default) 동기적으로
커밋레코드를 써야 한다면 지금 커밋레코드를 쓰기 요청한다. 이제 커밋레코드를
동기적/비동기적으로 썼든 간에 쓰기 요청한 커밋레코드의 완료를 대기한다.
그 다음 `JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT && JBD2_BARRIER` 인 경우
블럭레이어에 `WRITE_FLUSH` 요청을 보내고 완료대기 한다.

`t_forget` 리스트를 훑으면서 해당 `journal_head` 를 `t_checkpoint_list`
로 옮긴다. `journal_head->b_next_transaction` 이 비어있으면 해당 `journal_head`
를 `t_forget` 리스트에서 제거하고 레퍼런스카운트를 낮춘다. 이때 `buffer_head`
에 `BH_JBDDirty` 를 확인하여 `buffer_head` 에 `BH_Dirty` 를, `page` 에
`PG_dirty` 를, 패이지캐시트리에 `PAGECACHE_TAG_DIRTY` 를 설정하고 관련 inode 를
`backing_dev_info->wb.b_dirty` 리스트에 연결하여 writeback 대상이 되도록 한다.
(`__jbd2_journal_refile_buffer()`과 `mark_buffer_dirty()` 참고)
만일 `journal_head->b_next_transaction` 에 트렌젝션이 할당되어 있다면 해당
`journal_head` 가 커밋처리중에 새로운 트랜젝션의 대상이 되었다는 것으로
`journal_head->b_transaction` 에 `b_next_transaction` 을 할당하고 적합한 리스트
에 다시 연결한다. 보통은 BJ_Metadata 에 연결 되며 `b_next_transaction` 은 다시
`NULL` 이 된다. 이러한 경우 해당 `buffer_head` 및 `page 는 mm/fs 레이어에 dirty
상태로 환원되지않고 `BH_JBDDirty` 만 다시 설정된다.
커밋진행한 트랜젝션을 journal->j_checkpoint_transactions 에 연결 하여 체크포인
팅의 대상 트랜젝션이 되도록 한다.

### 6.1.7. T_CALLBACK Transaction State

    +- journal_t ---------------+
    | j_committing_transaction  | (= NULL)
    | j_commit_sequence         | <-+
    +---------------------------+   |
                                    |
    +- transaction_t ---+           |
    | t_tid             | ----------+
    +-------------------+

`j_committing_transaction` 에 `NULL` 을 할당하여 진행중인 커밋이 없음을 나타내
고 `j_commit_sequence` 을 커밋완료한 트랜젝션의 `t_tid` 로 갱신하여 커밋완료를
대기하는 프로세스가 깨어날 수 있도록 준비한다.
파일시스템에서 `journal_t->j_commit_callback` 을 등록했다면 해당 콜백을 호출한
다.

#### 6.1.8. T_FINISHED Transaction State

커밋한 트렌젝션의 journal_head 들이 모두 체크포인팅 완료되었다면 (다른 문맥에서
 동시적으로) 해당 트랜젝션을 `journal->j_checkpoint_transactions` 리스트에서
제거한 후, 트랜젝션자체를 해제한다.
`journal_t->j_wait_done_commit` 에서 대기중인 프로세스를 깨워 특정 트랜젝션의
커밋 완료를 기다리는 프로세스를 깨운다.

## 6.2. Commit Layout [section-commit-layout]

    -+- +- journal_header_t ----+
     ^  | h_magic               | JBD2_MAGIC_NUMBER
     |  | h_blocktype           | JBD2_DESCRIPTOR_BLOCK
     |  | h_sequence            | commit_transaction->t_tid
     |  +-----------------------+
     |  | UUID                  | journal_t->j_uuid (16bytes)
     |  +-----------------------+
     |  +- journal_block_tag_t -+
     |  | t_blocknr/_high       | buffer_head->b_blocknr
     |  | t_flags               | JBD2_FLAG_SAME_UUID
    4k  +-----------------------+
     |  +- journal_block_tag_t -+       .
     |  | ...                   |       .
     |  +-----------------------+       .
     |  +- journal_block_tag_t -+       .
     |  | ...                   |       .
     |  +-----------------------+       .
     |  +- journal_block_tag_t -+
     |  | t_blocknr/_high       | buffer_head->b_blocknr
     |  | t_flags               | JBD2_FLAG_SAME_UUID | JBD2_FLAG_LAST_TAG
     v  +-----------------------+
    -+- +- ORIGINAL FS DATA ----+
     ^  |.......................| Original Filesystem meta/data
     |  |.......................|       .
    4k  |.......................|       .
     |  |.......................|       .
     v  |.......................|       .
    -+- +-----------------------+       .
     ^  +- ORIGINAL FS DATA ----+       .
     |  |.......................|       .
     .  |.......................|       .
     |  |.......................|       .
     v  |.......................|       .
    -+- +-----------------------+
    -+- +- journal_header_t ----+
     ^  | h_magic               | JBD2_MAGIC_NUMBER
     |  | h_blocktype           | JBD2_DESCRIPTOR_BLOCK
     |  | h_sequence            | commit_transaction->t_tid
     |  +-----------------------+
     |  +- UUID ----------------+
     |  +- journal_block_tag_t -+
    ... +- journal_block_tag_t -+
     |  +           .           +
     |  +           .           +
     |  +           .           +
     |  +           .           +
     |  +- journal_block_tag_t -+
     |  +- journal_block_tag_t -+
     v  | t_flags               | JBD2_FLAG_SAME_UUID | JBD2_FLAG_LAST_TAG
    -+- +-----------------------+
     ^  +- ORIGINAL FS DATA ----+
     |  +- ORIGINAL FS DATA ----+
     v  +- ORIGINAL FS DATA ----+
    -+- +-----------------------+
     ^  +- commit_header -------+
     |  | h_magic               | JBD2_MAGIC_NUMBER
     4k | h_blocktype           | JBD2_COMMIT_BLOCK
     |  | h_sequence            | commit_transaction->t_tid
     v  | h_commit_sec/nsec     | now.tv_sec/nsec
    -+- +-----------------------+

> * Each block might be discrete.
>     * See `jbd2_journal_next_log_block(journal_t, unsigned long long)`
> * If the number of meta/data blocks is big enough, another descriptor might
>   be neccessary.

# 7. Checkpoint Transactions [section-checkpoint-trasaction]

---

    jbd2__journal_start(journal_t, blocks)
        start_this_handle(journal_t, handle)
            add_transaction_credits(journal_t, blocks)
                if (jbd2_log_space_left(journal) < jbd2_space_needed(journal))
                    __jbd2_log_wait_for_space(journal_t)
                        jbd2_log_do_checkpoint(journal);

저널영역의 로그는 그 영역이 부족할 때까지 체크아웃되지 않으며 저널요청을 시작하
는 쪽에서 이러한 경우를 확인하고 체크포인팅한다.

> NOTE
> 체크포인팅으로 빠지지 않는 다른 흐름도 있으니 그러한 경우는 코드를 살표보라.

    jbd2_log_do_checkpoint(journal_t)
        jbd2_cleanup_journal_tail(journal)

        transaction = journal->j_checkpoint_transactions

    restart:
        while (transaction->t_checkpoint_list)
            if (!buffer_dirty(bh))
                if (__jbd2_journal_remove_checkpoint(jh))
                    return
                continue

            __buffer_relink_io(jh)
                transaction->t_checkpoint_io_list = jh
            __flush_batch(journal_t, &batch_count)
                write_dirty_buffer(bh, WRITE_SYNC)
                    if (!test_clear_buffer_dirty(bh))
                        return
                    submit_bh(WRITE_SYNC, bh)
            goto restart

> NOTE
> psuedo 코드이며 자세한 흐름은 코드 참고

    +- transaction_t -------+
    | t_checkpoint_list     | xxx
    | t_chechpoint_io_list  | <-----+
    +-----------------------+       |
                                    |
    +- journal_head ----+-----------+           ...             ...
    | b_cp_transaction  | (= NULL)  |
    | b_cpnext          | <---------|---------- ...
    | b_transaction     | (= NULL)  |
    | b_bh              | <-+       |
    +-------------------+   |       |
                            |       |
    +- buffer_head -+-------+       |           ...             ...
    | b_private     | <-------------+
    | b_jlist       | (= BJ_None)
    | b_state       | (= BH_Dirty)
    | b_blocknr     | (original target block)
    | b_page        | <-+
    +---------------+   |
                        |
    +- page ----+-------+                       ...             ...
    | flags     | (= PG_dirty)
    | mapping   | <---------+
    +-----------+           |
                            |
    +- address_space ---+---+                   ...             ...
    | page_tree         | (= PAGECACHE_TAG_DIRTY)
    | host              | <-+
    +-------------------+   |
                            |
    +- inode ---+-----------+                   ...             ...
    | i_state   | (= I_DIRTY_PAGES)
    | i_wb_list | --------------+
    +-----------+               |
                                |
    +- backing_dev_info ----+   |               ...             ...
    | wb.b_dirty            | <-+
    +-----------------------+

checkpoint 단계에서는 `transaction->t_checkpoint_list` 에 연결된 원래
파일시스템에 직접 쓰였어야 할 `buffer_head` 들이 연결되어 있는데 마지막까지 이
`buffer_head` 가 `BH_Dirty` 상태일 경우에만 해당 `buffer_head` 에 대한 I/O 를
요청하며 `transaction->t_checkpoint_list` 에서 제거한다. 만일 dirty 가 아니면
I/O 또한 요청하지 않고 단순히 리스트에서 제거하는 동작만 수행한다. 이러한
경우는 저널 커밋후 dirty 가 된 버퍼/페이지들을 writeback 루틴에서 이미 디스크에
I/O 요청을 보낸 경우로 이미 clean 한 상태로 바뀐 경우이다.

    restart2:
        while (transaction->t_checkpoint_io_list)
            if (buffer_locked(bh))
                wait_on_buffer(bh)
                goto restart2
            __jbd2_journal_remove_checkpoint(jh)
        ...
        jbd2_cleanup_journal_tail(journal)

`t_checkpoint_io_list` 에 연결해둔 I/O 요청을 완료한 `buffer_head` 들을
대상으로 I/O 완료를 대기한다. 완료된 `buffer_head` 들을 대상으로 체크포인트
관련 리스트에서 제거한다.
마지막으로 `jbd2_cleanup_journal_tail()` 를 호출하여 불필요한 블럭을 반환한다.

    +- transaction_t -------+
    | t_checkpoint_list     | xxx
    | t_chechpoint_io_list  | <-----+
    +-----------------------+       |
                                    |
    +- journal_head ----+-----------+           ...             ...
    | b_cp_transaction  | (= NULL)  |
    | b_cpnext          | <---------|---------- ...
    | b_transaction     | (= NULL)  |
    | b_bh              | <-+       |
    +-------------------+   |       |
                            |       |
    +- buffer_head -+-------+       |           ...             ...
    | b_private     | <-------------+
    | b_jlist       | (= BJ_None)
    | b_state       |
    | b_blocknr     | (original target block)
    | b_page        | <-+
    +---------------+   |
                        |
    +- page ----+-------+                       ...             ...
    | flags     |
    | mapping   | <---------+
    +-----------+           |
                            |
    +- address_space ---+---+                   ...             ...
    | page_tree         |
    | host              | <-+
    +-------------------+   |
                            |
    +- inode ---+-----------+                   ...             ...
    | i_state   |
    | i_wb_list | xxx
    +-----------+

체크포인트가 완료되면 관련 `buffer_head`, `page`, `inode` 가 clean 한 상태가
되어 writeback 대상에서 제외된다.

# 8. Revocation [section-revoke]

TODO

# 9. Escaping [section-escape]

TODO
