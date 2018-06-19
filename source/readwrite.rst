Read-Write set semantics - 读写集语义
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This documents discusses the details of the current implementation about
the semantics of read-write sets.

Transaction simulation and read-write set - 交易模拟和读写集（read-write set）
'''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

During simulation of a transaction at an ``endorser``, a read-write set
is prepared for the transaction. The ``read set`` contains a list of
unique keys and their committed versions that the transaction reads
during simulation. The ``write set`` contains a list of unique keys
(though there can be overlap with the keys present in the read set) and
their new values that the transaction writes. A delete marker is set (in
the place of new value) for the key if the update performed by the
transaction is to delete the key.

- 在背书节点上的交易模拟期间会产生一个交易的read-write set。`read set`包含在模拟期间交易读取到的唯一key及对应version。`write set`交易改写的唯一key（可能与`read set`中的key重叠）及对应的新value。如果交易的更新操作是删除一个key，则在`write set`为该key设置一个delete标记。

Further, if the transaction writes a value multiple times for a key,
only the last written value is retained. Also, if a transaction reads a
value for a key, the value in the committed state is returned even if
the transaction has updated the value for the key before issuing the
read. In another words, Read-your-writes semantics are not supported.

- 此外，如果交易中对一个key改写多次，则只保留最后的修改值。如果交易中读取一个key的值，即使交易在读取之前更新了该key的值，读取到的也会是之前提交过的而不是刚更新的。换句话说，不能读取到同一交易中修改的值。

As noted earlier, the versions of the keys are recorded only in the read
set; the write set just contains the list of unique keys and their
latest values set by the transaction.

- 如前所述，key的version只记录在`read set`；`write set`只包含key及对应新value。

There could be various schemes for implementing versions. The minimal
requirement for a versioning scheme is to produce non-repeating
identifiers for a given key. For instance, using monotonically
increasing numbers for versions can be one such scheme. In the current
implementation, we use a blockchain height based versioning scheme in
which the height of the committing transaction is used as the latest
version for all the keys modified by the transaction. In this scheme,
the height of a transaction is represented by a tuple (txNumber is the
height of the transaction within the block). This scheme has many
advantages over the incremental number scheme - primarily, it enables
other components such as statedb, transaction simulation and validation
for making efficient design choices.

- 对于`read set`的version的实现有很多种方案，最基本要求就是为key生成一个非重复标识符。例如用递增的序号作为version。在目前的代码实现中我们使用了blockchain height作为version方案，就是用交易的height作为该交易所修改的key的version。交易height由一个结构表示（见下面Height struc），其中TxNum表示这个tx在block中的height（译注：也就是交易在区块中的顺序）。该方案相较于递增序号有很多优点–主要是这样的version可以很好地利用到诸如statedb、交易模拟和校验这些模块中。

Following is an illustration of an example read-write set prepared by
simulation of a hypothetical transaction. For the sake of simplicity, in
the illustrations, we use the incremental numbers for representing the
versions.

::

    <TxReadWriteSet>
      <NsReadWriteSet name="chaincode1">
        <read-set>
          <read key="K1", version="1">
          <read key="K2", version="1">
        </read-set>
        <write-set>
          <write key="K1", value="V1"
          <write key="K3", value="V2"
          <write key="K4", isDelete="true"
        </write-set>
      </NsReadWriteSet>
    <TxReadWriteSet>

Additionally, if the transaction performs a range query during
simulation, the range query as well as its results will be added to the
read-write set as ``query-info``.

Transaction validation and updating world state using read-write set
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

A ``committer`` uses the read set portion of the read-write set for
checking the validity of a transaction and the write set portion of the
read-write set for updating the versions and the values of the affected
keys.

In the validation phase, a transaction is considered ``valid`` if the
version of each key present in the read set of the transaction matches
the version for the same key in the world state - assuming all the
preceding ``valid`` transactions (including the preceding transactions
in the same block) are committed (*committed-state*). An additional
validation is performed if the read-write set also contains one or more
query-info.

This additional validation should ensure that no key has been
inserted/deleted/updated in the super range (i.e., union of the ranges)
of the results captured in the query-info(s). In other words, if we
re-execute any of the range queries (that the transaction performed
during simulation) during validation on the committed-state, it should
yield the same results that were observed by the transaction at the time
of simulation. This check ensures that if a transaction observes phantom
items during commit, the transaction should be marked as invalid. Note
that the this phantom protection is limited to range queries (i.e.,
``GetStateByRange`` function in the chaincode) and not yet implemented
for other queries (i.e., ``GetQueryResult`` function in the chaincode).
Other queries are at risk of phantoms, and should therefore only be used
in read-only transactions that are not submitted to ordering, unless the
application can guarantee the stability of the result set between
simulation and validation/commit time.

If a transaction passes the validity check, the committer uses the write
set for updating the world state. In the update phase, for each key
present in the write set, the value in the world state for the same key
is set to the value as specified in the write set. Further, the version
of the key in the world state is changed to reflect the latest version.

Example simulation and validation
'''''''''''''''''''''''''''''''''

This section helps with understanding the semantics through an example
scenario. For the purpose of this example, the presence of a key, ``k``,
in the world state is represented by a tuple ``(k,ver,val)`` where
``ver`` is the latest version of the key ``k`` having ``val`` as its
value.

Now, consider a set of five transactions ``T1, T2, T3, T4, and T5``, all
simulated on the same snapshot of the world state. The following snippet
shows the snapshot of the world state against which the transactions are
simulated and the sequence of read and write activities performed by
each of these transactions.

::

    World state: (k1,1,v1), (k2,1,v2), (k3,1,v3), (k4,1,v4), (k5,1,v5)
    T1 -> Write(k1, v1'), Write(k2, v2')
    T2 -> Read(k1), Write(k3, v3')
    T3 -> Write(k2, v2'')
    T4 -> Write(k2, v2'''), read(k2)
    T5 -> Write(k6, v6'), read(k5)

Now, assume that these transactions are ordered in the sequence of
T1,..,T5 (could be contained in a single block or different blocks)

1. ``T1`` passes validation because it does not perform any read.
   Further, the tuple of keys ``k1`` and ``k2`` in the world state are
   updated to ``(k1,2,v1'), (k2,2,v2')``

2. ``T2`` fails validation because it reads a key, ``k1``, which was
   modified by a preceding transaction - ``T1``

3. ``T3`` passes the validation because it does not perform a read.
   Further the tuple of the key, ``k2``, in the world state is updated
   to ``(k2,3,v2'')``

4. ``T4`` fails the validation because it reads a key, ``k2``, which was
   modified by a preceding transaction ``T1``

5. ``T5`` passes validation because it reads a key, ``k5,`` which was
   not modified by any of the preceding transactions

**Note**: Transactions with multiple read-write sets are not yet supported.

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/

