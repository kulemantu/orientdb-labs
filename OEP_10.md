#Summary:

Disk-based transactions with record/index key granularity.

##Goals:

Create transaction processing engine which supports following features:

1. Log records are created during transaction processing should be logged on disk not into memory to reduce memory consumption and avoid a risk of OOM and data corruption.
2. On transaction level, all locks should have record/key level granularity, not component granularity.
3. On level of single component operation (for example during insertion of (key, rid) pair into sb-tree) locks should have page granularity (in the example above some pages of sb-tree will be locked till insertion of (key, rid) pair in progress), not component granularity.

##Non-Goals

Improve speed of component operations in single-thread mode.

##Success metrics:

Improved scalability of massive write operations on performance benchmarks both in transactional and nontransactional modes.

##Motivation:

1. Almost all modern databases (MySQL, PostgreSQL, etc.) work with concurrent operations on record/key level granularity of locks.
Also, our investigation of results of performance benchmarks shows that component level locks are not enough to achieve good scalability of writes.
2. Keeping of changes of all pages only inside of RAM causes conspicuous memory consumption and may lead to OOM and data corruption.
3. During normal transaction processing we do not apply changes directly but collect all changes inside of atomic operation
(so-called deferred updates), and if we need to read changed pages, we apply those changes back to pages. Such approach increase system fragility. Any error inside of WAL or atomic operation will lead to data corruption. 
4. Applying of changes to the pages at every read operation decreases the speed of read of affected pages at ten times as result even read a single page which is changed inside of transaction will affect the speed of the whole transaction.

##Description:

###High level design

All changes are performed on pages are logged inside of WAL. Each log record consist of two parts: first part contains information which is needed to restore page content after the crash (redo part), the second part contains information which is needed to rollback page changes (undo part). Redo part is stored in the form of binary diff between original and changed binary presentations of a page.

Proposed transaction protocol allows using fine granularity locks during transaction processing by using a fascinating feature of all our data structures.  When we store/delete entry inside of any data structure
it has a page which satisfies the following requirement - if all changes are made on data structure but this single page (domain page) is not updated then data structure still is in the valid state, but 
data are treated as absent inside of given data structure. 

Each data structure changes may be split on two parts "structure modification changes" and "logical changes." 

Let's consider for example tree based index. When we insert (key, rid) pair inside of a tree, we make such changes as a split of parent nodes and a split of leaf page itself. All those changes are "structure modification changes." 
As a final step, we should add (key, rid) pair inside of the leaf page (which plays a role of domain page).
Till this entry is add to the leaf page (key, rid) pair is still absent in database, but tree structure is completely valid.
Once we put (key, rid) pair inside leaf page (execute "logical" insertion of data inside tree) it will be treated as stored inside 
of database.

In every index which we use leaf, page plays a role of domain page. For clusters, such domain pages are pages of
cluster transaction map.

To restore data after the system crash, we replay all transaction changes from the log till the point of crash and then 
revert all uncompleted transactions.

When transaction rollback is performed, we read WAL records from the last record logged into the WAL till the first record
and rollback changes are logged into those records using information is stored in undo part of WAL record.

During rollback it is possible to perform two types of rollbacks "page level" rollback when changes performed on the page reverted from this page during rollback and "logical" rollback when the action which is opposite to the executed logical operation will be performed. Logical rollbacks are performed only to revert logical changes and page level rollbacks are performed on structure modification changes. 

Every logical rollback changes again logged into the WAL. It will allow restoring data in case of system crash.

There are several cases of processing of rollbacks and data restore operations.

1. If rollback happens in the middle of structure modification changes 
we will rollback all changes applied to the pages on binary level and at the end of rollback 
the data structure will look like it was at the begging of a transaction.
2. If we rollback transaction after logical change is done, we will perform action on data structure which compensates executed action  instead of rolling back of structure modification changes, so at the end of rollback data structure will *logically* look like 
it was at the begging of the transaction.

Taken all above into account it becomes obvious that to implement "isolation" feature of transaction it is enough to keep only 
key/record level locks during a transaction and keep only page level locks inside of structure modification changes.

So what about restore of a system after the crash. How to keep data in a correct state even if the system crashed during transactional rollback.
To make it possible, we introduce a new type of log record which is called compensation log record (CLR). 
This record keeps a pointer to the log record which should be rolled back next after the log record which
was rolled back by operation wich generates current CLR. 
Every time we perform a rollback of page changes are logged inside of log record we put related CLR record.

Such CLR record contains only redo information. 
Redo information may include following data:

1. During rollback of a structure modification changes CLR redo part contains log record undo part. 
2. If we perform a logical rollback, then redo part is empty. 
3. If we complete structure modification changes before applying of changes to the domain page we also add CLR record with empty redo part which points to the last record logged before the first structure modification changes to a record.
The presence of such CLR record forces the rollback procedure to skip structure modification
changes during rollback of the transaction once they are completed and changes of domain page are logged.

There are several variants of restoring of database content after system crash:

1. A system is crashed inside of execution of structure modification changes. 
There are no any CLRs, so we merely restore data structure till the point when a system was crashed. 
And then rollback all page changes and restore initial state of the data structure before a transaction.
2. A system is crashed during rollback of structure modification changes. 
In such case we restore all structure modification changes, then by applying of CLR records, we will repeat partial rollback and by examinating of a content of last CLR record, we will find which pages still should be rolled back and will restore initial state of the data structure before a transaction.
3. Structure modification changes are completed, and CLR record is put at the end of this changes. 
In such case, we restore all structure modification changes and by processing of CLR record will rollback all changes which exist *after* structure modification changes but will not rollback structure modification changes itself. There is a good reason why we can not rollback structure modification changes. 
Page locks are released once we apply structure modification changes, and those pages may be changed by other successfully committed transactions. At the end of data, restore procedure changed data structure will be logically in the same state as it was before a transaction is started.
4. Structure modification changes and changes on domain page are completed in such case we will rollback only changes of domain page
and skip structure modification changes because of a presence of CLR record. We do not perform logical remove of data but only revert content of domain page because of concurrent access to the data structure it may be in the invalid state till complete restore of a state 
of all transactions will be completed.
At the end of data restore procedure changed data structure will be logically in the same state as it was before a transaction is started.

As you can see above if we restore system after crash, some space may be lost at the end of data restore procedure but amount of 
such space should be so minor that may be not taken into account.

What is interesting that even if we implement only new transaction processing protocol but will not change 
lock model of already existing components we still will increase system speed and scalability.
Let's suppose we have two ongoing transactions on the system, and component lock model is used.
First transaction changes components A and B and second transaction changes components A and C. 
In the current implementation, those transactions will be serialized but in the new implementation, transactions will go in parallel once one of them completes changes in component A. 

So at a first stage, we may implement protocol itself and then change lock model of components from component level to page level in next releases (such models will be described soon in separate OEPs if this OEP will be approved).

Let's consider the rest two operations on data structures - update and delete.
 
During execution of delete operation it is executed by following steps:

1. Delete an entry from domain page.
2. Put a request on the сlean up a queue to clean up consumed space after tx commit (it is needed only for a cluster). 

So during transaction rollback or restore, we revert only domain page content and as a result, a record is automatically restored.

Consider in details second item of delete algorithm - cleanup queue. 
When we delete record in a cluster, it is reasonable to claim space consumed by data back to the storage manager.

To make that possible, we create a cleanup queue which is filled by operations performed during a transaction (delete/update) which contains position to the record to be cleared  (this position is always the same even during internal page defragmentation and will not be changed after the record delete).
If a transaction is rolled back then, the cleanup queue is cleared, and no changes will be performed. If a transaction is committed then, changes are applied in a background thread. This clean up queue consumes very few memory, each entry queue consists of only two fields  
clusterId and position of record inside of data file, but it also may be bounded, we may allow adding no more than
1 000 000 of such entries at any time for a single transaction and 10 000 000 entries in total may be contained in the queue. The same limit may be applied to the amount of locks which may be acquired by the transaction to avoid any risk of OOM.
One of the ideas is to use ThreadPoolExecutor with a bounded queue and a maximum number of threads equal to an amount of cores.

Cleanup threads which process this queue will pull each entry and process it in a separate transaction.
So even if a system is crashed then the transaction will be rolled back, and a tiny amount of space will be lost.

The logic of update of data is similar because an update is a mix of deletion of previous data and an addition of new data.

1. Perform structure modification operations to add new data if needed (not needed for indexes, in the index, the only value inside of leaf page is updated).
2. Update domain page.
3. Put a request on the сlean up a queue to clean up consumed space after tx commit.  

So during rollback:
1. If rollback happens in the middle of execution of structure modification operations, then we revert all changes on binary level.
2. If rollback happens after domain page update, we remove new record data from a cluster and revert record pointer to old record content.

Restore logic is same as logic during rollback with the only exception that we do not remove already added record but revert domain page content. We do not perform logical remove of data but only revert content of domain page because of concurrent access to the data structure it may be in the invalid state till complete restore of a state of all transactions will be performed.

Once again it is worth to note that all those procedures will work even if we will use component locks, not page locks, providing a much better level of scalability without of changing of component implementation.

###Low-level design


##Alternatives:


##Risks and assumptions:


##Impact matrix

- [X] Storage engine
- [ ] SQL
- [ ] Protocols
- [ ] Indexes
- [ ] Console
- [ ] Java API
- [ ] Geospatial
- [ ] Lucene
- [ ] Security
- [ ] Hooks
- [ ] EE