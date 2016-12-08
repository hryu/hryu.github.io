---
layout: post
title: IO Scheduler Hooks Explained.md
---

## Contents

1. [Introduction][section-introduction]  
2. [Elevator Hooking Functions][section-elv-hooks]  
3. [Merge Sequence and Calling Hooks][section-merge-seq-call-hooks]  
3.1. [BIO Side Merge][section-bio-merge]  
3.1. [Request Side Merge][section-req-merge]  
4. [Request Movement between Containers][section-req-move]  
4.1 [Node and Containers][section-node-cont]  
4.1.1. [Request nodes][section-req-nodes]  
4.1.2. [Containers][section-containers]  
4.2 [Request Operation and Movement][section-req-op-mov]  
5. [APIs][section-apis]  
5.1. [Submit BIO][section-submit-bio]  
5.2. [Make Requests][section-make-req]  
5.2.1 [Attempts Merge BIO into Request on Plug List][section-attempt-plug]  
5.2.1.1 [See if BIO is Mergable into a Request][section-bio-mergealbe]  
5.2.1.2 [Merge BIO into Request][section-merge-bio-req]  
5.2.2 [Search Mergeable Cadidate with BIO in IO Scheduler]
      [section-merge-candidate]  
5.2.3 [Query Whether BIO Mergeable into Request][section-query-bio-merge]  
5.2.4 [Attempt Back for Front Merge][section-attempt-back-front-merge]  
5.2.4.1 [Merge Requests][section-merge-reqs]  
5.2.5. [Get a Request][section-get-req]  
5.2.6. [Add Requests to IO Scheduler][section-add-reqs]  
5.3. [Fetch Requests][section-fetch-req]  
5.3.1. [Typical Block Driver's Strategy Routine][section-strategy-routine]  
5.3.2 [Move Request to Dispatch Queue][section-move-dispatch-q]  
5.4 [Complete Requests][section-complete-req]


---

## 1. Introduction [section-introduction]

I/O Scheduler 에서 사용되는 hook 함수들에 대해 알아보고 `blk_queue_bio()` 과
블럭드라이버 처리 흐름속에서 각 hook 들이 호출되는 경로와 bio, request 들이
merge 되는 과정을 살펴본다.

> NOTE  
> 특정한 io scheduler 알고리즘에 대해서는 다루지 않는다.  

## 2. Elevator Hooking Functions [section-elv-hooks]

| index | return     | name                       | parameters                |
| ----- | ---------- | -------------------------- | ------------------------- |
| 1     | int        | elevator_merge_fn          | q \*, req \*\*, bio \*    |
| 2     | void       | elevator_merged_fn         | q \*, req \*, type        |
| 3     | void       | elevator_merge_req_fn      | q \*, req \*, req \*      |
| 4     | int        | elevator_allow_merge_fn    | q \*, req \*, bio \*      |
| 5     | void       | elevator_bio_merged_fn     | q \*, req \*, bio \*      |
|       |            |                            |                           |
| 6     | int        | elevator_dispatch_fn       | q \*, force               |
| 7     | void       | elevator_add_req_fn        | q \*, req \*              |
| 8     | void       | elevator_activate_req_fn   | q \*, req \*              |
| 9     | void       | elevator_deactivate_req_fn | q \*, req \*              |
|       |            |                            |                           |
| 10    | void       | elevator_completed_req_fn  | q \*, req \*              |
|       |            |                            |                           |
| 11    | request \* | elevator_former_req_fn     | q \*, req \*              |
| 12    | request \* | elevator_latter_req_fn     | q \*, req \*              |
| 13    | void       | elevator_init_icq_fn       | io_cq \*                  |
| 14    | void       | elevator_exit_icq_fn       | io_cq \*                  |
| 15    | int        | elevator_set_req_fn        | q \*, req \*, bio \*, gfp |
| 16    | void       | elevator_put_req_fn        | req \*                    |
| 17    | int        | elevator_may_queue_fn      | req \*, rw                |

1. bio 와 merge 될만한 request 를 iosched 내부에서 찾고 이를 merge 타입과
   함께 리턴.  
2. iosched 내부의 request 들에 대한 merge 작업이 완료된 후 iosched 에게 통보.  
   `attempt_front/back_merge()` 이후 성공하면 호촐됨.
   `attempt_front/back_merge()` 함수는 요청중인 request 와 인접한 request 를
   iosched 내부로 부터 얻기 위해 `elv_former/latter_request()` 를 호출하는데
   그 함수에서 반환한 request 와 요청중인 request 가 merge 될 수 있는지
   `attempt_merge` 를 호출하여 가능하면 merge 한다.  
   merge 하는 경우 `elevator_merge_req_fn` 를 호출한다.  
3. 두개의 request 를 merge 한 후에 호출되어 iosched 에게 통보.  
   `attempt_merge()` 에서 iosched 내부의 request 들이 merge 되면 호출된다.  
4. bio 를 request 에 merge 할 수 있는지 iosched 에게 질의.  
   request 는 iosched 내부에 존재하며 request 는 `q->last_merge` 이거나
   back merge 를 시도하기 위해 `elv_rqhash_find()` 호출하여 검색한 것이다.  
5. bio 를 request 에 merge 한 후 호출하여 iosched 에게 통보  
6. iosched 내부의 request 를 얻기위해 블럭드라이버의 전략루틴에서 호출
   iosched 내부의 request 를 삭제/추출한후 `request_queue->queue_head` 에
   추가하여 드라이버로 request 를 넘긴다. (`blk_fetch_request()`)  
7. `plug->list` 에서 request 를 삭제한 후, iosched 로 추가할 때 호출.  
   `blk_flush_plug_list()` 에서 호출되는데 `blk_flush_plug_list()` 함수는
   plug->list 에서 request 를 제거하고 `__elv_add_request(INSERT_SORT_MERGE)`
   를 호출한다. `__elv_add_request()` 함수는 `elv_attempt_insert_merge()` 를
   이용하며 먼저 기존 request 와 merge 를 시도하고 실패한 경우에만 해당
   request 를 elevator->hash 에 등록 후 add_req 를 호출하여 iosched 에게
   request 를 내부에 추가하도록 한다.  
8. 블럭드라이버로 request 를 넘길때 호출하여 iosched 에게 통보.  
   `blk_peek_request()` 를 이용하여 iosched 로부터 dispatch 를 호출, request 를
   iosched 내부에서 제거 및 획득 한후 `REQ_STARTED` 플래그를 확인하여
   블럭드라이버로 처음 넘기는 request 인 경우에 호출하여 iosched 로 통보한다.  
9. dispatch 한 request 를 request_queue 의 디스패치큐에 다시 삽입하는 경우에
   iosched 에게 통보.  
   이 함수는 `elv_requeue_request()` 함수내에서 실행되는데 `REQ_SORTED` 플래그
   를 확인하여 activation 된 request 에 대해 iosched 에게 통보하고
   `REQ_STARTED` 플래그를 해제한 후
   `__elv_add_request(ELEVATOR_INSERT_REQUEUE)` 를 호출하여 request 를 다시
   `request_queue->queue_head` 에 추가 및 `REQ_SOFTBARRIER` 플래그를 설정한다.
10. 블럭드라이버에서 request 의 처리 완료 iosched 에게 통보.  
    블럭드라이버는 처리완료된 request 에 대해 `blk_finish_request()` 를 호출함
    으로써 해당 request 의 처리완료를 iosched 에 알린다.  
11. iosched 내부의 request 에 대해 인접한 request 가 있는지 질의 및 획득  
    `elv_merge()` 를 이용하여 bio 를 iosched 내부의 request 와 merge 한 이후
    merge 되어 상태정보가 변경된 request 에 대해 인접한 내부 request 와 merge
    를 시도해보기 위해서 질의 및 획득한다.
12. 11 와 동일
13. io context for request_queue 생성시 iosched 에게도 초기화 기회 부여.  
    새로운 request 생성할때 request_queue 에 대한 icq 가 없는 경우에만 icq 를
    생성하고 초기화  
14. 13 과 반대로 icq 가 파괴될 때 iosched 에게 통보  
15. 생성한 request 에 대해 iosched 에게 내부 자료 구조 추가 작업 기회 부여.  
16. 파괴되는 request 에 대해 iosched 에게 내부 자료 구조 변경 작업 기회 부여.  
17. 새로운 request 를 생성하기전 iosched 에게 request 생성을 해도 되는지 질의  
    `get_request()` 함수에서 request 를 획득 시도할 때 iosched 에게 request
    생성 가능 여부를 질의하는데 생성할 수 없다면 기존 request 가 완료되어 사용
    가능해진때까지 대기하도록한다

---

## 3. Merge Sequence and Calling Hooks [section-merge-seq-call-hooks]
### 3.1 BIO Side Merge [section-bio-merge]

1. `bio_attempt_[front/back]_merge()`  
   (request on plug list)  
2. `elevator_allow_merge_fn()`  
3. 2 (true) -> `elevator_merge_fn()`  
4. 3 (true) -> `bio_attempt_[front/back]_merge()`  
   (request on iosched)  
   (`q->last_merge` or `q->hash` or `e->RBROOT` or `e->LIST`)  
5. 4 (true) -> `elevator_bio_merged_fn()`  
6. 5 (true) -> `elevator_[latter/former]_req_fn()` (true) ->  
   `elevator_merge_req_fn()`  
   (interaction with request)  

### 3.2. Request Side Merge [section-req-merge]

1. `elevator_init_icq_fn()` -> `elevator_set_req_fn()`  
2. 1 -> `elevator_merge_fn()` (merge with existing request on iosched)  
3. 2 (false) -> `elevator_add_req_fn()`  

4. 2/3 -> `elevator_allow_merge_fn()` (true) -> `elevator_merge_fn()` (true)  
   -> `elevator_bio_merged_fn()`  
   (interaction with bio)  
5. 4 (true) -> `elevator_[latter/former]_req_fn()` (true) ->  
   'elevator_merge_req_fn()` -> `elevator_put_req_fn()` ->  
   `elevator_merged_fn()`  

6. 2/3/4/5 -> `elevator_dispatch_fn()` -> `elevator_activate_req_fn()`  
7. 6 -> `elevator_deactivate_req_fn()` -> `elevator_dispatch_fn()` -> ...  
   (in case of requeue)  
8. 6 -> `elevator_completed_req_fn()` -> `elevator_put_req_fn()`  
9. `elevator_exit_icq_fn()`  

---

## 4. Request Movement between Containers [section-req-move]
### 4.1. Node and Containers [section-node-cont]
#### 4.1.1. Request nodes [section-req-nodes]

    request->queuelist
    request->hash
    request->rb_node

    request->cmd_flags

#### 4.1.2. Containers [section-containers]

    current->plug->list
    request_queue->queue_head
    elevator->hash
    elevator->elevator_data->RBROOT
    elevator->elevator_data->LIST

### 4.2. Request Operation and Movement [section-req-op-mov]

| operaion | node               | direction | container or flags              |
| -------- | ------------------ | --------- | ------------------------------- |
| plug     | request->queuelist | ADD_TO    | current->plug->list             |
|          |                    |           |                                 |
| get req  | request->queuelist | DEL_FROM  | current->plug->list             |
|          | request->cmd_flags | SET       | REQ_SORTED                      |
|          | request->hash      | ADD_TO    | elevator->hash                  |
|          | request->rb_node   | ADD_TO    | elevator->elevator_data->RBROOT |
|          | request->queuelist | ADD_TO    | elevator->elevator_data->LIST   |
|          |                    |           |                                 |
| add req  | request->queuelist | DEL_FROM  | current->plug->list             |
|          | request->cmd_flags | SET       | REQ_SORTED                      |
|          | request->hash      | ADD_TO    | elevator->hash                  |
|          | request->rb_node   | ADD_TO    | elevator->elevator_data->RBROOT |
|          | request->queuelist | ADD_TO    | elevator->elevator_data->LIST   |
|          |                    |           |                                 |
| merge    | next->queuelist    | DEL_FROM  | elevator->elevator_data->LIST   |
|          | next->rb_node      | DEL_FROM  | elevator->elevator_data->RBROOT |
|          | prev->hash         | DEL_FROM  | elevator->hash                  |
|          | prev->hash         | ADD_TO    | elevator->hash (repositopn)     |
|          | next->hash         | DEL_FROM  | elevator->hash                  |
|          |                    |           |                                 |
| dispatch | request->hash      | DEL_FROM  | elevator->hash                  |
|          | request->rb_node   | DEL_FROM  | elevator->elevator_data->RBROOT |
|          | request->queuelist | DEL_FROM  | elevator->elevator_data->LIST   |
|          | request->queuelist | ADD_TO    | request_queue->queue_head       |
|          | request->cmd_flags | SET       | REQ_STARTED                     |
|          |                    |           |                                 |
| requeue  | request->cmd_flags | CLEAR     | REQ_STARTED                     |
|          | request->cmd_flags | SET       | REQ_SOFTBARRIER                 |
|          | request->queuelist | ADD_TO    | request_queue->queue_head       |

---

## 5. APIs [section-apis]
### 5.1. Submit BIO [section-submit-bio]

    blk_start_plug()

    submit_bio(bio)
        generic_make_request(bio)
            generic_make_request_checks(bio)
            if (current->bio_list)
                bio_list_add(current->bio_list, bio)
                return
            current->bio_list = &bio_list_on_stack
            do
                q->make_request_fn(q, bio)
                bio = bio_list_pop(currnt->bio_list)
            while (bio)
            current->bio_list = NULL

    blk_finish_plug()

### 5.2. Make Requests [section-make-req]

    blk_queue_bio(q, bio)
        where = ELEVATOR_INSERT_SORT
        ...
        if (bio->bi_rw & (REQ_FLUSH | REQ_FUA))
            where = ELEVATOR_INSERT_FLUSH
            goto get_request

merge 될 수 없는 request 의 생성

        if (attempt_plug_merge(q, bio, &request_count))
            return

bio 가 머지될만한 request 를 `plug->list()` 에서 검색하고 가능하면 merge 한다.

        ret = elv_merge(q, &req, bio)                   [allow_merge, merge]
        if (ret == ELEVATOR_BACK_MERGE)
            if (bio_attempt_back_merge(q, req, bio))
                elv_bio_merged(q, req, bio);            [bio_merged]
                    if (!attempt_back_merge(q, req))    [latter_req, merge_req]
                        elv_merged_request(q, req, ret) [merged]
                    return
        else if (ret == ELEVATOR_FRONT_MERGE)
            if (bio_attempt_front_merge(q, req, bio))
                elv_bio_merged(q, req, bio);            [bio_merged]
                if (!attempt_front_merge(q, req))       [former_req, merge_req]
                    elv_merged_request(q, req, el_ret)  [merged]
                return

bio 와 merge 될만한 request 를 iosched 내부에서 검색하고 가능하면 merge 한다.
bio 가 merge 된 request 에 대해 다시 한번 iosched 내부의 request 들을 검색하여
가능하면 merge 한다.

        req = get_request(q, flags, bio)                [may_queue, init_icq, set_req]
        init_request_from_bio(req, bio)

새로운 request 를 생성하고 적절한 iosched hook 함수를 호출하여 통보한다.

        if (current->plug)
            if (request_count >= BLK_MAX_REQUEST_COUNT)
                blk_flush_plug_list(plug, false)    [merge_req]
                                                    [add_req, dispatch]
                                                    [activate_req]
                                                    [deactivate_req]
                                                    [completed_req, put_req]
            list_add_tail(&req->queuelist, &plug->list)

plug 가 설정된 경우 `request_count` 가 `BLK_MAX_REQUEST_COUNT` 를 넘지 않는 선
에서 `plug->list` 추가하고 리턴한다.

        else
            add_acct_request(q, req, where)
                blk_account_io_start(rq, true)
                __elv_add_request(q, req, where)
            __blk_run_queue(q)

plug 가 설정되지 않은 경우 `__elv_add_request(ELEVATOR_INSERT_SORT)` 를 호출
하여 iosched 내부에 request 를 추가하고 `q->request_fn()` (블럭드라이버 전략
루틴) 를 바로 호촐하여 처리를 시도한다.

#### 5.2.1 Attempts Merge BIO into Request on Plug List [section-attempt-plug]

    attempt_plug_merge(q, bio, &request_count)
        if (!current->plug)
            return false
        list_for_each_entry(rq, currnt->plug->list)
            lr = blk_try_merge(rq, bio)
            if (lr == ELEVATOR_BACK_MERGE)
                ret = bio_attemp_back_merge(q, req, bio)
            else if (lr == ELEVATOR_FRONT_MERGE)
                ret = bio_attempt_front_merge(q, req, bio)
        return true if merged

plug 리스트에 존재하는 req 에 대하여 반복을 하면서 해당 bio 와 합쳐질수 있는
인접한 sector 를 가진req 을 찾고 합칠 수 있다면 front/back 에 따라 해당 req 의
큐에 적절히 추가한다. io scheduler 와의 연결은 없다.

##### 5.2.1.1. See if BIO is Mergable into a Request [section-bio-mergealbe]

    blk_try_merge(request, bio)
        if (blk_rq_pos(rq) + blk_rq_sectors(rq) == bio->bi_sector)
            return ELEVATOR_BACK_MERGE;
        else if (blk_rq_pos(rq) - bio_sectors(bio) == bio->bi_sector)
            return ELEVATOR_FRONT_MERGE;
        return ELEVATOR_NO_MERGE;

sector 를 기반으로 bio 가 request 에 merge 될 수 있는지, 있다면 어떤 타입인지
반환

##### 5.2.1.2. Merge BIO into Request [section-merge-bio-req]

    bio_attempt_back_merge(q, req, bio)
        if (!ll_back_merge_fn(q, req, bio))
            return false
        req->biotail->bi_next = bio
        req->biotail = bio
        req->__data_len += bio->bi_iter.bi_size
        req->ioprio = ioprio_best(req->ioprio, bio_prio(bio))
        return true

request 의 bio 리스트의 마지막에 bio 를 추가하여 merge 함.

    bio_attempt_front_merge(q, req, bio)
        if (!ll_front_merge_fn(q, req, bio))
            return false
        ...
        bio->bi_next = req->bio;
        req->bio = bio
        req->buffer = bio_data(bio)
        req->__sector = bio->bi_iter.bi_sector
        req->__data_len += bio->bi_iter.bi_size
        req->ioprio = ioprio_best(req->ioprio, bio_prio(bio))
        return true

req 의 bio 리스트의 처음에 bio 를 추가하여 merge 함.

#### 5.2.2. Search Mergeable Cadidate with BIO in IO Scheduler [section-merge-candidate]

    elv_merge(q, req, bio)
        if (q->last_merge && elv_rq_merge_ok(q->last_merge, bio)) [allow_merge]
            ret = blk_try_merge(q->last_merge, bio)
            if (ret != ELEVATOR_NO_MERGE)
                *req = q->last_merge
                return ret
        __rq = elv_rqhash_find(q, bio->bi_sector)
        if (elv_rq_merge_ok(__rq, bio))                         [allow_merge]
            return ELEVATOR_BACK_MERGE
        return elv->elevator_merge_fn(q, req, bio)              [merge]

bio 와 합쳐질 수 있는 req 를 elevator 의 내부의 큐에서 찾고 type 과 함께 반환
하여 merge 시도에 사용될 수 있도록한다.  
캐쉬로 유지되는 `q->last_merge` 와 먼저 해당 bio 가 합쳐질 수 있는지 iosched
에게 질의(`elv_rq_merge_ok()`) 한다. 만일 합쳐질수 있다면 해당 req 와 type 을
반환. 캐쉬에서 찾을 수 없다면 elevator 에 유지되는 bi_sector 를 키로 하는
해쉬에서 req 를 찾아고 존재하면 해당 req 와 ELEVATOR_BACK_MERGE 를 반환.
두가지 모두 실패하면 `elevator_merge_fn()` 를 호출하여 iosched 내부에서 검색.

#### 5.2.3. Query Whether BIO Mergeable into Request [section-query-bio-merge]

    elv_rq_merge_ok(req, bio)
        elv_iosched_allow_merge(req, bio)
            return elv->elevator_allow_merge_fn(q, req, bio)    [allow_merge]

bio 가 request 에 merge 될 수 있는지 iosched 에게 질의

#### 5.2.4. Attempt Back and Front Merge [section-attempt-back-front-merge]

    attempt_back_merge(q, req)
        next = elv_latter_request(q, req)                       [latter_req]
        if (next)
            return attempt_merge(q, rq, next)                   [merge_req]
        return 0

    attempt_front_merge(q, req)
        prev = elv_former_request(q, req)                       [former_req]
        if (prev)
            return attempt_merge(q, prev, rq)                   [merge_req]
        return 0

`elv_merge()` 로 얻은 request 와 type 에 따라 동작하며
`bio_attemp_[front/back]_merge()` 로 인해 request 의 앞/뒤에 bio 가 새로 추가
되어 request 의 정보가 갱신되었으므로 갱신된 내용에 따라 request 끼리 merge 가
가능한지 여부를 다시 판단하고 경우에 따라 merge 함.

##### 5.2.4.1. Merge Requests [section-merge-reqs]

    attemp_merge(r, req, next)
        ...
        req->biotail->bi_next = next->bio
        req->biotail = next->biotail
        req->__data_len += blk_rq_bytes(next)

        elv_merge_requests(q, req, next)                        [merge_req]
        ...
        next->bio = NULL
        __blk_put_request(q, next)

merge 가 가능한 경우를 점검한 후, req ==> next 의 순으로 request 들을 merge
한다. merge 후 `elv_merge_requests()` 를 호출하여 elevator 가 갱신된 request 에
대한 추가 작업을 하게 함. (예, next request 를 elevator 내부 관리 큐에서 삭제)

#### 5.2.5. Get a Request [section-get-req]

    get_request(q, rw_flags, bio, gfp)
        rl = blk_get_rl(q, bio)
        rq = __get_request(rl, rw_flags, bio, gfp)
            elv_may_queue(q, rw_flags)                          [may_queue]
            ...
            if (blk_rq_should_init_elevator(bio) && !blk_queue_bypass(q))
                rw_flags |= REQ_ELVPRIV
                ...
            ...
            rq = mempool_alloc(rl->rq_pool, gfp_mask)
            blk_rq_init(q, rq)
            blk_rq_set_rl(rq, rl);
            rq->cmd_flags = rw_flags | REQ_ALLOCED
            if (rw_flags & REQ_ELVPRIV)
                ...
                if (!icq && ioc)
                    icq = ioc_create_icq(ioc, q, gfp)           [init_icq]
                elv_set_request(q, rq, bio, gfp)                [set_req]
                ...
            ...
        ...
     init_request_from_bio(req, bio)
        req->cmd_type = REQ_TYPE_FS
        req->cmd_flags |= bio->bi_rw & REQ_COMMON_MASK
        req->errors = 0
        req->__sector = bio->bi_iter.bi_sector
        req->ioprio = bio_prio(bio)
        blk_rq_bio_prep(req->q, req, bio)
            rq->cmd_flags |= bio->bi_rw & REQ_WRITE
            ...
            rq->bio = rq->biotail = bio
            ...
     ...

새로운 request 를 할당/초기화하고 `elv_set_request()` 를 호출하여 elevator 가
새로 생성된 request 에 추가 작업을 하도록 허용

#### 5.2.6. Add Requests to IO Scheduler [section-add-reqs]

    blk_flush_plug_list(plug, from_schedule)
        flush_plug_callbacks(plug, from_schedule)
        list_splice_init(&plug->list, &list)
        list_sort(NULL, &list, plug_rq_cmp)
        while (!list_empty(&list))
            rq = list_entry_rq(list.next)
            list_del_init(&rq->queuelist)
                ...
                if (rq->cmd_flags & (REQ_FLUSH | REQ_FUA))
                    __elv_add_request(q, rq, ELEVATOR_INSERT_FLUSH)
                else
                    __elv_add_request(q, rq, ELEVATOR_INSERT_SORT_MERGE)
                                                        [merge_req, add_req]
        if (q)
            queue_unplugged(q, depth, from_schedule)
            if (from_schedule)
                blk_run_queue_async(q)
            else
                __blk_run_queue(q)
                    __blk_run_queue_uncond(q)
                        q->request_fn(q)

plug 리스트에서 iosched 내부의 자료구조로 request 를 추가한다. 경우에 따라
이미 내부에 존재하는 request 와 merge 될 수도 있고 새롭게 추가될 수도 있다.

    __elv_add_request(q, rq, where)
        if (rq->cmd_flags & REQ_SOFTBARRIER)
            ...
        else if (!(rq->cmd_flags & REQ_ELVPRIV) &&
                  (where == ELEVATOR_INSERT_SORT ||
                   where == ELEVATOR_INSERT_SORT_MERGE))
            where = ELEVATOR_INSERT_BACK

        ...
        switch (where)
            case ELEVATOR_INSERT_REQUEUE
            case ELEVATOR_INSERT_FRONT
                rq->cmd_flags |= REQ_SOFTBARRIER
                list_add(&rq->queuelist, &q->queue_head)
            case ELEVATOR_INSERT_BACK
                rq->cmd_flags |= REQ_SOFTBARRIER
                elv_drain_elevator(q)                               [dispatch]
                list_add_tail(&rq->queuelist, &q->queue_head)
                __blk_run_queue(q)
            case ELEVATOR_INSERT_SORT_MERGE
                if (elv_attempt_insert_merge(q, rq))                [merge_req]
                    break
            case ELEVATOR_INSERT_SORT
                rq->cmd_flags |= REQ_SORTED
                ...
                q->elevator->type->ops.elevator_add_req_fn(q, rq)   [add_req]
            case ELEVATOR_INSERT_FLUSH
                rq->cmd_flags |= REQ_SOFTBARRIER
                blk_insert_flush(rq)

where 에 따라 다른 작업을 하도록 구성됨.  
`ELEVATOR_INSERT_SORT_MERGE` 로 호촐
된 경우에는 먼저 기존 request 들과 요청중인 request 를 merge 하도록 시도하고
실패한 경우에만 새롭게 추가한다.  
`ELEVATOR_INSERT_REQUEUE` 로 호출된 경우에는 request 를 다시 `q->queue_head` 에
삽입하여 다음 `elv_next_request()` 시 블럭드라이버가 해당 request 를 가져가도록
한다.

### 5.3. Fetch Requests [section-fetch-req]
#### 5.3.1. Typical Block Driver's Strategy Routine [section-strategy-routine]

    do
        ...
        req = blk_fetch_request(q)                  [dispatch, activate_req]
        if (req)
            ...
        else
            ...
            schedule()
        ...
    while (1)

`blk_fetch_request()` 를 반복호출하여 요청이 반환된 경우 해당 요청을 처리하고
없으면 잠든다.

#### 5.3.2. Move Request to Dispatch Queue [section-move-dispatch-q]

    blk_fetch_request(q)
        rq = blk_peek_request(q)                    [dispatch, activate_rq]
        if (rq)
            blk_start_request(rq)
                blk_dequeue_request(req)
                    list_del_init(&rq->queuelist)
        return rq

    blk_peek_request(q)
        while ((rq = __elv_next_request(q)) != NULL)    [dispatch]
            ...
            if (!(rq->cmd_flags & REQ_STARTED))
                if (rq->cmd_flags & REQ_SORTED)
                    elv_activate_rq(q, rq)              [activate_req]
                rq->cmd_flags |= REQ_STARTED
            ...
            ret = q->prep_rq_fn(q, rq)
            ...
        return rq

    __elv_next_request(q)
        while (1)
            if (!list_empty(&q->queue_head))
                return list_entry_rq(q->queue_head.next);
            ...
            if (unlikely(blk_queue_bypass(q)) ||
                !q->elevator->type->ops.elevator_dispatch_fn(q, 0)) [dispatch]
                return NULL

    blk_start_request(req)
        blk_dequeue_request(req)
            list_del_init(&req->queuelist)
            ...
        ...
        blk_add_timer(req)

iosched 내부의 자료구조로부터 request 를 삭제하여 획득하고 해당 request 를
디스패치큐인 `q->queue_head` 에 추가하여 드라이버가 request 를 가져가도록 한다.
디바이스가 가져가는 경우 `q->queue_head` 에서 삭제된다.

### 5.4. Complete Requests [section-complete-req]

    blk_end_request(req, err, nr_bytes)
        blk_end_bidi_request(req, err, nr_bytes, 0)
            if (blk_update_bidi_request(req, err, nr_bytes, bidi_bytes))
                return true
            blk_finish_request(req, err)
            return false

request 에 처리할 데이터가 남았으면 true, 모두 처리하였으면 false

    blk_update_bidi_request(rq, error, nr_bytes, bidi_bytes)
        if (blk_update_request(rq, error, nr_bytes))
            return true
        ...
        return false

    blk_update_request(req, error, nr_bytes)
        ...
        while (req->bio) {
            bio = req->bio;
            bio_bytes = min(bio->bi_iter.bi_size, nr_bytes);
            if (bio_bytes == bio->bi_iter.bi_size)
                req->bio = bio->bi_next;
            req_bio_endio(req, bio, bio_bytes, error);
                bio_advance(bio, nbytes)
                if (bio->bi_size == 0 && !(rq->cmd_flags & REQ_FLUSH_SEQ))
                    bio_endio(bio, error)
                        ...
                        bio->bi_end_io(bio, error)
                        ...
            total_bytes += bio_bytes;
            nr_bytes -= bio_bytes;
            if (!nr_bytes)
                break

        if (!req->bio)
            ...
            return false
        ...
        return true

    blk_finish_request(req, err)
        blk_delete_timer(req)
        if (req->end_io)
                req->end_io(req, error);
        else
            ...
            __blk_put_request(req->q, req);

    __blk_put_request(req->q, req)
        ...
        elv_completed_request(q, req)                       [completed_req]
        if (req->cmd_flags & REQ_ALLOCED)
            flags = req->cmd_flags;
            rl = blk_rq_rl(req)
            blk_free_request(rl, req)
                if (rq->cmd_flags & REQ_ELVPRIV)
                    elv_put_request(rl->q, rq)              [put_req]
                ...
            freed_request(rl, flags)
            blk_put_rl(rl)

request 의 처리가 모두 완료된 경우에 `elv_completed_request()` 이 호출되어
request 의 처리가 완료되었음을 iosched 에게 통보한다.
