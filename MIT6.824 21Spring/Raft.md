---
sort: 3
---

# Raft

This part is all based on the paper [Raft](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf), which is absolutely worth carefully reading.

## What is Raft

Raft is a consensus algorithm for managing a replicated log. Compared with PAXIOS, it is much easier to understand, and also provides powerful functions to support consistency of managed logs, as well as fault tolerance.

## Roles in Raft

Raft is implemented on several distributed severs to provide functions. There are three kinds of roles defined in Raft:

1. **Leader**: it is used to manage logs on itself as well as on all other servers. Services would communicate with it to send logs and apply states. Logs of the Raft leader are all valid, and conflicts on other servers, if any, are invalid. Besides communicate with services, it also communicates with other servers to synchronize its logs. The final goal is to reach an agreement: all the logs on different servers are the same, and the states applied on the state machine are the same, which is called *committed* logs.
2. **Follower**: it is mainly used as replicate of leader. Imagine that: if there were only one server, the leader, logs and states could also be valid and recorded correctly without failures beyond exceptions. To avoid this, replicates are very useful and necessary, because we need the availability. Followers all keep a replicate of logs on leader locally, and once the leader crashes, one of the followers would become next leader.
3. **Candidate**: just as described above, once the leader crashes, a follower would become the next leader, but not any of followers could become leader. A follower wishing to become leader shall become candidate firstly, admitted by the majority of all the servers then, and finally it could become the leader. The middle status of follower and leader is called candidate.

The conversion rule between the three roles is simple: follower could convert to candidate, and candidate could also convert back to follower; candidate could convert to leader, but leader could only convert to follower.

What's more, we can see that Raft is not a purely distributed consistency algorithm, because it still needs a leader. But on the other hand, this leader is not selected manually on purpose, but  selected from a group of similar servers. It is kind of centric, but not totally.

## Rules for Vote

### Introduction

The conversion from follower to leader has rule, which is called **vote**.

The background is that the leader would always send heartbeat message to all the followers regularly, but if the follower does not hear from the leader for a while, it believes the leader is dead, and a new leader must be elected. At this time, this follower would convert to **candidate**. Notice: this conversion is not driven by someone external, but the follower itself, with the help of a timer inside.

The new candidate then sends vote request to all other servers. Other servers, after receiving the vote request, must determine whether it should grant vote to it. So there requires a detailed rule. This rule must guarantee:

1.  The candidate's logs should be as new and complete as the receiver's
2. The receiver has not voted for other candidates after the former leader crashed

In this way, we introduces a concept named **term**, to indicate the which leader hosts; and the **index** of log is also recorded to show how new log the server contains. By following the consistency principle, we could determine a log by term and index, and the first critical rule is:

**A log is identified by its term and index**

This means by confirming a pair of term and index, we can identify a certain single log, and no other possibilities. In this way, we could let the candidate send vote request with these messages, and receivers would compare it with its own. To be more specific, the last log of its record, this is because of the second critical rule:

**All logs are ordered by its index in ascending order**

This is guaranteed by the fact that each time the leader receives a new log, it appends the log to its record with index incremented, and because of consistency, the index increments regularly, even on different leaders in different time. Because each leader would guarantee the rule from its start point, and former logs are also guaranteed by former leaders.

If the last log of the receiver is newer than the candidate according to the message received, it proves that the receiver contains logs that do not exist on candidate, and the candidate should not be leader in fact: we should let the follower with newest, which is equal to the longest on logs' length, logs become the leader.

Another point is to make it apart from former vote, because the receiver of course voted for someone in last term. In this way, the candidate should increase its current term, and includes this information in the vote request, and receivers would take it into consideration. The receiver would not vote for any others if it has voted for someone. This is called *first-come-first-serve* principle.

The vote request and respond could be like:

```go
// vote request
{
    Term int			// candidate’s term
    CandidateId int		// candidate requesting vote
    LastLogIndex int	// index of candidate’s last log entry
    LastLogTerm int		// term of candidate’s last log entry
}

// vote response
{
    Term int 			// currentTerm, for candidate to update itself
    voteGranted bool 	// true means candidate received vote
}
```

rules:

> 1. Reply false if term < currentTerm
> 2.  If votedFor is null or candidateId, and candidate’s log is at least as up-to-date as receiver’s log, grant vote

There are some variables saved in server's state, and you could refer to the paper for more details.

### Lab 2A

Lab 2A is based on vote, and the goal is to implement Raft's vote mechanism. It must work correctly in different scenarios, including initial setup, leader crashes, network failures and so on.

As I have described above, the initial setup could be easy. But to avoid split vote, which means a lot of followers become candidates and start up votes at the same time, and it may not make a candidate win a majority of servers to elect a leader, the expiration time, which means the threshold of not receiving heartbeat message from leader, should be randomized. And then, it reduces the possibility of followers staring up votes simultaneously.

By comparing the term of  vote request with their own, receivers could know whether this request is out of date or a valid one. This also applies to later other RPCs.

Once a leader has been elected, it should send heartbeat message to all other servers to avoid another election starting. For the heartbeat message, the same check would be applied, and expired heartbeat message should not reset the timer of election timeout. If the message received contains a term newer than receiver's own, the receiver would update its current term. 

In this way, as we can see, this becomes a loop: with the help of vote rules, each time the leader absent, a new leader with the most complete logs would be elected and work as its former, as long as there are more than a half of servers are still alive, for example, 3 of 5.

These rules also guarantee that each leader must be in different terms in an ascending order, though inconsistent perhaps. Because in one term, all servers would be synchronized to the same term. Even some servers' term fails to be updated to the newest term, in the next round of election, the term contained in vote request would only allows those candidates having the newest term to be elected. As for the lagging servers, even if they start an election, their term is out of date and index lagging than others' in fact, and it would be impossible for them to be elected as the leader. So that Raft's term would never roll back but increase forever.

## Rules for Log replication

The replication process of logs is the most important part in Raft, and the most complicated part. The definition of this process looks smooth and reasonable, but it becomes pretty hard for implementation, because a lot of corner cases must be taken into consideration.

In this part, I would go through from the very start status, also the most ideal status.

### Ideal status

Leader elected after startup send log append RPC to followers successfully without any delay. In this way, the replication on followers could always be synchronized with leader. In this scenario, everything works smoothly.

### If leader crashes

Leader could crash in the middle of synchronization, and in this scenario, a new leader should be elected immediately. And because of all followers' logs are the same as former leader's, nothing else is needed.

### If delay exists

If there are some delays between leader and followers, things could be complicated. As we discussed above, only followers with newest log could be elected as new leader. And even the new leader has been elected, it still needs to figure out each followers' status, like which log should be sent to them separately. 

To apply this, each leader should maintain two data structures: an array of next index of log of each follower, and an array of index of log that could confirmed being replicated on each follower. When initialized, the records of `nextIndex` array should all be assigned as leader's commit index + 1, and `lastApplied` should be all set as 0. This difference indicates that the records in `nextIndex` are only estimations of followers' status, or am eager guess; and for records in `lastApplied`, they are solid and reliable. These records should both be updated by later communication between leader and followers.

The follower should also check the validation of each log it receives, there are some steps:

1. Locate the index used to mark start point. The leader would send two variables: `preLogTerm` and `preLogIndex`, to record the term and index of the start log's preceding log.
2. After finding out the preceding log by index, the follower should check whether the term of this log is the same as `preLogTerm`. Because one log could be identified by term and index, once the two variables agree, the follower could confirm the log needing appended is valid and should be accepted. But if the check fails, it should be rejected.
3. The logs are then appended after the preceding log.

### If quick synchronization required

We could also implement the feature that leader could send a batch of logs to followers for replication. And in this way, followers should go though the log array received, and check each log's validation. But be careful to avoid out of index error when implementing.

### If network delay exists

A common delay is just lagging of leader's status, but network delay is a real headache. Imagine that: some RPCs are sent in an order but received in a different order. If the follower does not check, a disaster would be caused finally, and it must be a total chaos.

#### For Follower:

##### When receiving request in an earlier term

The follower should refuse the request directly with its current term assigned respond.

##### When receiving request in a later term

The follower should update its own term as new as the request's, and the check for the log append is still necessary.

##### When receiving request with conflict preceding log information

The follower should delete logs from the conflict point.

##### When receiving request with existing logs

Sometimes followers could receive requests containing logs already existing in their records, and their job is to check these log, if conflicts happen, they should truncate their own records from the conflicting point.

#### For leader:

##### When receiving respond with a later term

This indicates the leader is out of date and it should immediately covert back to a follower.

##### When receiving respond with an earlier term

This indicates the respond is out of date, and the leader should ignore it and never update its records according to the respond. In fact, to guarantee leader's safety, the leader should **confirm the terms of the request it sent, the respond it received, and its own current term are the same, and only then should it accepts the respond as a valid one**.

##### When receiving respond with success flag

The follower then should update its records of this follower sent the respond according to the message. For example, it should update the `nextIndex` and `lastApplied` for the follower, and currently, the leader just set the `nextIndex` value as the last log's index it sent to the follower in this communication plus 1, and `lastApplied` value as the same as this last log's index.

### Optimization

There are some optimizations possible for Raft, and one has been mentioned above, that is to send multi logs in a single RPC, instead of one by one. Another is used when conflict happens, the follower should not just reply with a false flag, but it should provide the conflict log's term, as well as the first index of this very **term** it stores. In this way, by searching the term in its own log records, the leader could find out the preceding log of this whole term's logs, and use its information as `preLogIndex` and `preLogTerm`, and send logs following it the next time. In this way, the leader could bypass a whole term in one single communication, instead of one index per communication. But once the leader fails to find logs with this term, it should set the `nextIndex` value of this follower as conflict index directly.

## Rules for Commit

Once a log has been applied to the state machine, it is called *committed*, and it should be determined by the leader. The leader does it by confirming this very log has been replicated on a majority of servers (including itself), and this is to avoid losing committed logs. Once a log has been replicated on a majority of servers, it would never be lost unless the whole Raft system crashes.

When the leader has determined the log should be applied and committed, it increase its `commitIndex` variable to the index of the log. In practice, it is equal to plus 1 to it. For followers, they should check the `commitIndex` in log append RPC it received from leader, and if this value is larger than its own, the follower knows it should commit this log, too.

But it is possible that the follower does not contain the log indicated by the leader's `commitIndex`, then the follower should compare the `commitIndex` from leader, and the index of the last log appended in this RPC. The follower would choose the smaller one as the target it should commit itself to.

## Lab 2B & Lab 2C

In the two labs, our job is to implement a fully functional Raft with fault-tolerance ability, mainly I described above. The key points I have also discussed above, but during the process of implementation,  I did not figure it out so well, and I had to struggle with a lot of wired errors and bugs. To be honest, however, some bugs looked wired at first glance, but by analyzing the detailed log with code logic, finally I could find out why this happened and how to solve them. In this way, a detailed and well-formatted logs are very important and necessary, and I organized it like this:

```
[server 1, term=1, commit idnex=1] something described here
```

This helped me a lot when testing, and I also attached a complete log of running Lab 2 [here](./lab2-result.txt). This log file contains more than 30000 lines of printed logs, and several additional logs indicating testing procedures I added in test files.

Another lesson is that we must check the validation of requests and responds both. At first, I only checked when receiving requests, but not for responds, and finally outdated replies ruin all the system.

Again, the checking rule:

**request's term = respond's term = server's current term**

This could be another critical rule.

A big challenge of this series labs is the random failure when testing. Because there are a bunch of random-based tests, your code could pass in one round bur fail in another, and multi tests are required to test the correctness of your code. In this way, I wrote a bash script to help this process. It could run tests in parallel by running multi tasks at the same time, and it could record how many times your code fails in these tests, and persist the printed log of this round. I attached it below:

```shell
#!/bin/bash
count=0
rm *.txt
for ((i=0; i<$1; i+=$3))
do
    echo "$count/$1"

    for ((j=1; j<=$3; j++))
    do {
        filename="res-$j.txt"
        go test -run $2 -race > $filename
        current=$(grep -o 'ok' $filename |wc -l)
        num=$[i+j]
        if [ $current -gt 0 ]; then
            echo "($num) test $2 passed once"
        else
            echo "($num) !!!error happened when running test $2!!!"
            newFilename="error-$num.txt"
            mv $filename $newFilename
        fi
    } & 
    done
    wait

    count=$((${count} + $3))
    
done
echo "$2 tests finished: $count/$1"
failed=$(ls error*.txt |wc -l)
echo "test failed: $failed/$1"
```

This shell script should be used like:

```bash
./run_multi.bash 500 2C 5 2>/dev/null
```

It means running test 2C 500 times in total, and 5 tests in batch in parallel, and redirect warning flow to black hole (hide warning messages). This script would also copy logs of failed tests and store them at current directory for later check.

A reminder: do not make your computer or laptop turn into sleep mode while running tests, because the test could not guarantee return back to normal after being waken up.

### Milestone: Universal AppendEntry RPC

A milestone of my implementation is that I made to combine different kinds of AppendEntry RPCs as one single universal RPC. The three kinds of RPCs are: (1) heartbeat message, (2) log append message and (3) commit message. The heartbeat message is used to carry on heartbeat between leader and followers; the log append message is used to send entries, or logs to followers from the leader; the last one is used by the leader to inform followers a certain log has been committed by it. These implementations should not divergence at very first, but in fact, my implementation moved forward step by step with the Lab's requirements. In Lab 2A, only heartbeat message fashion was introduced; in Lab 2B and 2C, I found it could be impossible to guarantee consistency by using normal log append message to make followers commit logs only, because some logs could not be committed before the leader receives the last log appended and forwards it to followers with commitment information, so that I introduced a separated commitment message fashion besides normal log append message fashion.

The divergence of the last two message fashions is mainly because of the heartbeat message design. In fact, the heartbeat message should carry leader's commitment information, which was missed in my former design, and it had to be made up by a special-designed commitment message. It is a bad design, though, even we ignore the ugly divergence, because the commitment message could be lost during transition or for follower crashes.

After I figured these out, I could feel that I have gained a deeper understanding of Raft, and by struggling and debugging for a while, I successfully combined these three kinds of message fashions as one. Just as I described above, the heartbeat message is not only used as a signal, but also carries on commitment information. With help of this, the normal log append message would not worry about data lose and related commitment transition failures. Followers would also no longer maintain different functions dealing with different messages, but one in total, and just keeping an eye on validation check of AppendEntry RPC.

To be honest, the right implementation is always elegant, while the defective implementation looks ugly and redundant. 

## Rules for Snapshot and Lab 2D

To reduce the size of logs in one server, each server could use snapshot machinasm to store past logs, which have been confirmed by services that these logs have been applied indeed. The basic rules have been settled in Section 7 in the paper, but in Lab 2D of 2021, the guide is far from enough. I guess it is because this lab part was just added this year, and former labs have been arranged years ago with detailed guideline from students' experience. I have to admit that I spent a lot of time trying to understand the goals and designs of this lab, and I believe it would be helpful to record them here:

### The structure of Snapshot in Lab 2D

The snapshot should not only include logs to be snapshotted, but the value of the last command in the snapshot. In fact, the raw data of snapshot should contain:

1. **Snapshot**: logs to be put in the snapshot data
2. **v**: the last command value in these logs

The test program would use filed *v* to fetch the last command as a mark, so never forget it, though it could have little use for you.

### Do not block apply channel

In our lab, Raft uses a channel named `applyCh` to communicate with service (single direction, from Raft to service). In Lab 2D, after each message received from `applyCh`, the tester would determine whether logs should be trimmed and generate a new snapshot. In this way, it would invoke `Snapshot` function in our code. We must be careful about this, because the test program would wait forever until the `Snapshot` function returns, and following message sent to the channel would be blocked. **Never design the Snapshot function as a synchronization function**, unless you can guarantee that the function would be able to be called successfully after **each** log committed. 

In our former design, we discussed that a server could commit its logs in batch, and we would than naturally lock these codes when commit from log *I* to log *I+n*. And as for the commitment, we should guarantee the ascending order of log commitments, so that we could not release the lock before processing these commitments. An effective way is to realse the lock and wait for snapshot signal, but it could be hard to implement, because we could not guarantee there would not be inserted other operations in these commitments, and any side-effects in potential. So I just simply put snapshot operation in a single go rountine, and it would wait for the finish of message sending of the commitment, and start doing snapshot then.

### Remember to check Snapshot before accepting it

The snapshot is sent from leader, and the follower should accept it, but checks are necessary: the follower should check whether the snapshot is expired or behind its current logs by comparing the term field in requests and its own, as well as `lastIncludeIndex` field with its own and `commitIndex`. The last two rules, however, have not been mentioned in the paper, but it comes naturally: The follower should refuse older snapshot, as mentioned in the Lab 2D's description, and the most obvious way is to compare the `lastIncludeIndex`, because it indicates the end point of snapshot received and the follower's current snapshot. As for the comparsion between snapshot's `lastIncludeIndex` and follower's `commitIndex`, it is because after installing the snapshot, the follower would apply the snapshot to the service, and it could be pointless and incorrect to apply snapshot behind current commit index, which is equal to commit older logs that have already been committed again.

### Figure out when to apply snapshot

The operation, install snapshot, could be confusing because it is not finsihed in one single function or communication round. There would be four steps:

1. The follower receives the snapshot installation request, and then it checks the validation of the snapshot
2. The follower applies the valid snapshot to service by sending the snapshot to it via channel
3. The service then invokes `CondInstallSnapshot` of the follower to send the snpshot back to it to inform the follower that it has successfully applied the snapshot
4. The follower could finally apply the snapshot ot itself, modify its metadata, trim its logs and other operations if needed

This workflow is described in Lab 2D, but hard to understand at very first, and I believe it is because the action of installing snapshot is not finished by Raft, but by the service, and Raft only records the result of installation. That is why only after receiving signal from the service, the follower could apply the snapshot to itself (in fact, it should not be called apply, but store the snapshot in its own space).

### The modification of log location rules

Because there exists snapshot and logs could be trimmed, we need to calculate the real index of target log in leader and follower'a storage. 

- For leader: If the index calculated of target log is negative, it proves that it does not contain the log in its alive recording, and the log is stored in its snapshot. In this way, the leader should not send `AppendEntry` RPC, but `InstallSnapshot` RPC instead.
- For follower: If the follower receives an `AppendEntry` RPC, it should covert the `PrevLogIndex` of request to the real index in its recording. If it is neagative, then the follower should reject this RPC and indicate the leader to relocate to its alive logs.

Other operations, like looking for certain logs when conflicting happens, should also take the snapshot into consideration. I would set these index as `lastIncludeIndex` realted values if a search goes beyond the start point of alive logs.

## Lab running results

I ran Lab 2A for 3000 times, Lab 2B for 2000 times, Lab 2C for 500 times, Lab 2D for 200 times, and Lab 2 for 100 times as a whole in total. My program passed **every test**, which makes me believe the correctness of my program, I also noticed that there are some blogs on the Internet to declare that they passed all tests in most times and only failed in several times. I have to say these scenarios indicate that their implementations could have something wrong. Some others mentioned they broke some non-mandatory rules. For example, one blogger said he modified the interval of sending heartbeat message from 100 milliseconds required in the Lab's description to a smaller value to pass tests. I would not call it a correct implementation. It proves that the requirements of Lab 2 is carefully designed and could be met within the design of Raft described in the paper and hints on the Lab's page.

The student's guide posted by a former TA is also helpful, but as I mentioned above, it helps little to the newly added Lab 2D in 2021. I hope my blog can help you debug and pass all the tests. Good luck!