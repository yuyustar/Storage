## 1集群运行

```
//rpc
mkdir cmake-build-debug
cd cmake-build-debug
cmake ..
make
//raft集群
mkdir cmake-build-debug
cd cmake-build-debug
cmake..
make
```

![image-20240326163525697](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240326163525697.png)

## 2函数实现

RAFT算法实现函数：

领导选举流程

| 函数                               | 作用                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| void Raft::electionTimeOutTicker() | 负责查看是否该发起选举，如果该发起选举就执行doElection发起选举。 |
|                                    | 用一个死循环和计时器来实现                                   |
| void Raft::doElection()            | 实际发起选举，构造需要发送的rpc，并多线程调用sendRequestVote处理rpc及其相应。 |
|                                    | 具体实现过程见代码，或见3.1部分代码流程介绍                  |
| bool Raft::sendRequestVote()       | 负责发送选举中的RPC，在发送完rpc后还需要负责接收并处理对端发送回来的响应 |
|                                    | 具体实现过程见代码，或见3.2部分代码流程介绍                  |
| void Raft::RequestVote()           | 接收别人发来的选举请求，主要检验是否要给对方投票             |
|                                    | 具体实现过程见代码，或见3.3部分代码流程介绍                  |
| void Raft::leaderHearBeatTicker()  | 负责查看是否该发送心跳了，如果该发起就执行doHeartBeat。      |
|                                    | 其基本逻辑和选举定时器electionTimeOutTicker一模一样，不一样之处在于设置的休眠时间不同，这里是根据HeartBeatTimeout来设置，而electionTimeOutTicker中是根据getRandomizedElectionTimeout() 设置 |
| void Raft::doHeartBeat()           | 实际发送心跳，判断到底是构造需要发送的rpc，并多线程调用sendRequestVote处理rpc及其相应。 |
|                                    | 具体实现过程见代码，或见3.4部分代码流程介绍                  |
| bool Raft::sendAppendEntries       | 负责发送日志的RPC，在发送完rpc后还需要负责接收并处理对端发送回来的响应。 |
|                                    | 具体实现过程见代码，或见3.5部分代码流程介绍                  |
| void Raft::AppendEntries()         | 接收leader发来的日志请求，主要检验用于检查当前日志是否匹配并同步leader的日志到本机。 |
|                                    | 具体实现过程见代码，或见3.6部分代码流程介绍                  |
| 其他辅助代码流程详解               | 待完善                                                       |

## 3代码流程梳理

#### 3.1void Raft::doElection()

{

1. `lock_guard<mutex> g(m_mtx);`：使用C++11的RAII特性来自动管理锁的获取和释放。这样可以避免忘记释放锁导致的死锁问题。
2. 状态检查：如果当前节点的状态不是Leader，那么节点将开始选举过程。
3. 状态更新：将节点的状态设置为Candidate（候选人），并且当前的任期（term）加一。每个Raft节点在开始选举时都会增加自己的任期。
4. 自我投票：节点给自己投票，这是为了确保节点在选举中至少有一票。
5. `persist();`：用于将当前的状态持久化到存储中，以便在节点崩溃后能够恢复状态。
6. 重置选举定时器：记录下当前的时间，用于重置选举定时器。
7. 发布RequestVote RPC：对每个节点（除了自己）发送RequestVote RPC请求，请求它们投票给自己。这里使用了lambada来避免在新线程中获取锁，调用sendRequestVote()。
8. 线程创建和分离：为每个节点创建一个新的线程来发送RequestVote RPC请求，并通过`t.detach();`分离线程，这样主线程不会等待这些RPC请求的完成。
9. 请求参数初始化：为RequestVote RPC请求设置必要的参数，包括当前任期、候选人ID（即自己）、最后一条日志的索引和任期

}

#### 3.2bool Raft::sendRequestVote()

{

1. RPC调用：通过`m_peers[server]`获取对应服务器的代理对象，并调用其`RequestVote`方法发送投票请求。如果RPC调用失败（`ok`为`false`），则直接返回失败状态。
2. 处理响应：如果RPC调用成功，接下来会根据收到的响应来更新自己的状态。
3. 任期更新：如果响应中的任期（`reply->term()`）大于当前节点的任期（`m_currentTerm`），则当前节点需要更新自己的任期，并将自己设置为跟随者状态，同时将`m_votedFor`设置为-1（表示当前任期没有投票），然后持久化这些状态变更。
4. 任期不变或更小：如果响应中的任期小于或等于当前节点的任期，且投票被授予（`reply->votegranted()`为`true`），则当前节点将增加自己的投票计数（`votedNum`）。
5. 成为领导者：如果投票计数达到或超过集群中一半以上的节点（`votedNum >= m_peers.size()/2+1`），当前节点认为自己已经赢得了选举，将状态设置为领导者（`m_status = Leader`），并重置`votedNum`为0。
6. 初始化领导者状态：作为新选举出的领导者，节点需要初始化一些状态，如`nextIndex`和`matchIndex`，这些都是用于后续日志复制和一致性保证的。
7. 宣告领导者状态：创建并分离一个新的线程来执行`doHeartBeat`函数，该函数可能是用来向集群中的其他节点发送心跳消息，宣告自己为新的领导者。
8. 持久化状态：在状态变更后，调用`persist`函数来持久化状态，确保即使节点失败，也能恢复到正确的状态

}

#### 3.3void Raft::RequestVote()

{

1. 持久化操作：定义了一个`Defer`对象`ec1`，它将在`lock_guard`析构时（即互斥量解锁后）执行持久化操作。这是为了确保在处理完RequestVote请求后，节点的状态能够被持久化存储。

2. 任期比较：对请求中的任期（`args->term()`）与当前节点的任期（`m_currentTerm`）进行比较，根据不同的情况执行不同的操作：

   如果请求的任期小于当前节点的任期，说明请求已经过时，回复拒绝投票。

   如果请求的任期大于当前节点的任期，当前节点更新自己的任期，变为跟随者状态，重置投票记录，并回复投票。

3. 日志一致性检查：如果请求的任期与当前节点相同，接下来会检查请求中的日志条目是否是最新的。如果请求中的日志条目比接收者的日志旧，那么拒绝投票。

4. 投票检查：检查当前节点是否已经投票给其他节点。如果已经投票，并且不是给当前请求的候选人，那么拒绝投票。

5. 授予投票：如果所有检查都通过，当前节点同意投票给候选人，将`m_votedFor`设置为候选人的ID，并更新选举定时器。然后，设置回复为投票被授予。

}

#### 3.4 void Raft::doHeartBeat()

{

1. 锁定互斥量：使用`std::lock_guard<mutex> g(m_mtx);`来确保对共享资源的访问是线程安全的。
2. 检查领导者状态：如果当前节点的状态是领导者（`m_status == Leader`），则执行心跳发送逻辑。
3. 创建心跳计数智能指针：`auto appendNums = std::make_shared<int>(1);` 创建一个共享指针，用于计数正确响应心跳的节点数量。
4. 遍历节点：对于`m_peers`中的每个节点（除了自己，`if(i == m_me)`），执行以下操作：
   - 如果`m_nextIndex[i]`小于或等于`m_lastSnapshotIncludeIndex`，表示该节点的日志已经被快照覆盖，需要发送快照而不是心跳。创建并分离一个新的线程来发送快照。
   - 如果不需要发送快照，构造一个`AppendEntriesArgs`对象，准备发送心跳消息。
   - 获取上一条日志的索引和任期（`getPrevLogInfo`），用于心跳消息的构造。
   - 如果`preLogIndex`不是快照的索引，添加从`preLogIndex`的下一条日志开始的所有日志条目到心跳消息中。
   - 初始化心跳响应的共享指针`appendEntriesReply`。
5. 发送心跳：创建并分离一个新的线程来发送心跳消息给每个跟随者节点。线程函数是`sendAppendEntries`，它将处理心跳消息的发送和响应。
6. 更新心跳时间：在发送完所有心跳消息后，更新`m_lastResetHearBeatTime`为当前时间，用于记录下一次心跳的时间。

}



#### 3.5 bool Raft::sendAppendEntries

1. RPC调用：通过`m_peers[server]`获取对应服务器的代理对象，并调用其`AppendEntries`方法发送心跳请求。如果RPC调用失败（`ok`为`false`），则直接返回失败状态。

2. 处理响应：如果RPC调用成功，接下来会根据收到的响应来更新自己的状态。

3. 任期检查：无论响应如何，首先检查响应中的任期（`reply->term()`）：

   如果响应中的任期大于当前节点的任期，当前节点需要更新自己的任期，并变为跟随者状态，同时重置投票记录。

   如果响应中的任期小于当前节点的任期，这通常不应该发生，因此直接返回成功状态。

4. 领导者状态检查：如果当前节点不是领导者（`m_status != Leader`），则不处理响应，直接返回成功状态。

5. 处理成功响应：如果AppendEntries请求成功（`reply->success()`为`true`），则：

   增加接收心跳的节点数量（`appendNums`）。

   更新`m_matchIndex`和`m_nextIndex`，这些索引用于跟踪日志复制的进度。

   如果大多数节点（包括当前节点自己）已经接收了心跳，可以提交日志（`*appendNums >= 1 + m_peers.size()/2`）。

   更新`m_commitIndex`，这是Raft算法中的一个重要指标，表示已知被提交的日志的最大索引。

6. 处理失败响应：如果AppendEntries请求失败（`reply->success()`为`false`），则：

   更新`m_nextIndex`，根据跟随者的反馈调整下一次发送日志的起始索引。

7. 幂等性保证：在提交日志时，只有当当前term有日志提交时，才会更新`m_commitIndex`。这是为了确保领导者的完整性（Leader Completeness），即领导者拥有所有之前被提交的日志。

   `if (*appendNums >= 1 + m_peers.size()/2) { //可以commit了`            

   //两种方法保证幂等性，1.赋值为0 	2.上面≥改为==             

   `*appendNums = 0;  //置0`



#### 3.6void Raft::AppendEntries

1. **锁定互斥量**：使用`std::lock_guard<std::mutex> locker(m_mtx);`来确保对共享资源的访问是线程安全的。
2. **任期检查**：检查请求中的任期（`args->term()`）与当前节点的任期（`m_currentTerm`）：
   - 如果请求的任期小于当前节点的任期，说明请求已经过时，设置响应为失败，并更新响应中的任期为当前节点的任期。
   - 如果请求的任期大于当前节点的任期，当前节点需要更新自己的任期，并变为跟随者状态，同时重置投票记录。
   - 如果任期相等，继续处理请求。
3. **持久化操作**：定义了一个`Defer`对象`ec1`，它将在`lock_guard`析构时（即互斥量解锁后）执行持久化操作。这是为了确保在处理完AppendEntries请求后，节点的状态能够被持久化存储。
4. **状态更新**：如果当前节点是候选人（candidate）并且收到了同一个任期的领导者的AppendEntries请求，它需要转变为跟随者（follower）状态，并重置选举超时定时器。
5. **日志匹配**：检查请求中的日志条目是否与当前节点的日志匹配：
   - 如果请求中的`prevLogIndex`大于当前节点的最后一个日志索引，说明请求的日志已经过时，设置响应为失败，并更新`updatenextindex`为当前节点的最后一个日志索引加一。
   - 如果`prevLogIndex`小于快照索引（`m_lastSnapshotIncludeIndex`），说明请求的日志太旧，设置响应为失败，并更新`updatenextindex`为快照索引加一。
   - 如果日志匹配，复制请求中的日志条目到当前节点的日志中，并更新`m_commitIndex`。然后设置响应为成功。
6. **日志不匹配**：如果日志不匹配，需要找到不匹配的第一个日志条目，并更新`updatenextindex`为该日志条目的索引加一。然后设置响应为失败。
7. 寻找不匹配的第一个日志条目：接下来，代码通过一个循环从`prevLogIndex`开始向下遍历跟随者本地的日志，直到`m_lastSnapshotIncludeIndex`（快照包含的最后一个日志索引）。在循环中，跟随者检查当前索引位置的日志条目的任期（`getLogTermFromLogIndex(index)`）是否与请求中提供的前一个日志条目的任期（`getLogTermFromLogIndex(args->prevlogindex())`）相同。如果不同，说明找到了不匹配的日志条目。一旦找到不匹配的日志条目，跟随者将`updatenextindex`设置为找到的不匹配日志条目的索引加一`（index + 1）`，这告诉领导者从下一个索引开始重新发送日志条目。这样做是为了减少RPC调用的次数，优化日志复制的效率。





