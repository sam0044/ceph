======================
 MDS Config Reference
======================

.. confval:: mds_cache_mid
.. confval:: mds_allow_batched_ops
.. confval:: mds_dir_max_commit_size
.. confval:: mds_dir_prefetch_backend
.. confval:: mds_dir_prefetch_backend_max
.. confval:: mds_dir_max_entries
.. confval:: mds_decay_halflife
.. confval:: mds_beacon_interval
.. confval:: mds_beacon_grace
.. confval:: mon_mds_blocklist_interval
.. confval:: mds_reconnect_timeout
.. confval:: mds_tick_interval
.. confval:: mds_dirstat_min_interval
.. confval:: mds_scatter_nudge_interval
.. confval:: mds_client_prealloc_inos
.. confval:: mds_early_reply
.. confval:: mds_group_commit_enable
.. confval:: mds_group_commit_max_entries
.. confval:: mds_group_commit_max_interval
.. confval:: mds_default_dir_hash
.. confval:: mds_log_skip_corrupt_events
.. confval:: mds_bal_sample_interval
.. confval:: mds_bal_replicate_threshold
.. confval:: mds_bal_unreplicate_threshold
.. confval:: mds_bal_split_size
.. confval:: mds_bal_split_rd
.. confval:: mds_bal_split_wr
.. confval:: mds_bal_split_bits
.. confval:: mds_bal_merge_size
.. confval:: mds_bal_interval
.. confval:: mds_bal_fragment_interval
.. confval:: mds_bal_fragment_fast_factor
.. confval:: mds_bal_fragment_size_max
.. confval:: mds_bal_idle_threshold
.. confval:: mds_bal_max
.. confval:: mds_bal_max_until
.. confval:: mds_bal_mode
.. confval:: mds_bal_min_rebalance
.. confval:: mds_bal_overload_epochs
.. confval:: mds_bal_min_start
.. confval:: mds_bal_need_min
.. confval:: mds_bal_need_max
.. confval:: mds_bal_midchunk
.. confval:: mds_bal_minchunk
.. confval:: mds_replay_interval
.. confval:: mds_shutdown_check
.. confval:: mds_thrash_exports
.. confval:: mds_thrash_fragments
.. confval:: mds_dump_cache_on_map
.. confval:: mds_dump_cache_after_rejoin
.. confval:: mds_verify_scatter
.. confval:: mds_debug_scatterstat
.. confval:: mds_debug_frag
.. confval:: mds_debug_auth_pins
.. confval:: mds_debug_subtrees
.. confval:: mds_kill_mdstable_at
.. confval:: mds_kill_export_at
.. confval:: mds_kill_import_at
.. confval:: mds_kill_link_at
.. confval:: mds_kill_rename_at
.. confval:: mds_inject_skip_replaying_inotable
.. confval:: mds_kill_after_journal_logs_flushed
.. confval:: mds_wipe_sessions
.. confval:: mds_wipe_ino_prealloc
.. confval:: mds_skip_ino
.. confval:: mds_min_caps_per_client
.. confval:: mds_symlink_recovery
.. confval:: mds_extraordinary_events_dump_interval

Readdir page size
-----------------

The MDS encodes a readdir reply while holding ``mds_lock``, and the size of that
reply is chosen by the client. A client that asks for a whole directory -- or
that leaves the entry count unset, as libcephfs does -- is served the whole
directory in one reply, bounded only by a 512KB byte cap. Every other request on
that rank waits for the encoding to finish, so a single client walking a large
directory, such as a backup or an indexer, can dominate the latency everyone else
sees.

Readdir pages are already separate requests, so ``mds_lock`` is released between
them. Ending a page early therefore costs the reading client one more round trip
and gives the rank a point at which to serve the work waiting behind it.

By default the MDS does this only while other work is actually queued: while
encoding a page it examines the dispatch queue every
``mds_readdir_yield_entries`` entries, and ends the page once at least
``mds_readdir_yield_min_queued`` messages are waiting. A client walking a large
directory on an otherwise idle rank is left alone. The check interval is also the
smallest page the MDS will cut short to, so a busy rank still returns useful
pages rather than a handful of entries at a time. Set
``mds_readdir_yield_entries`` to 0 to disable this and restore the previous
behaviour.

.. confval:: mds_readdir_yield_entries
.. confval:: mds_readdir_yield_min_queued

Two unconditional caps are also available, and are off by default. These apply
whether or not the rank is busy, so a walker pays the extra round trips even when
nothing else is running; prefer the conditional bound above unless you need a
hard limit on reply size.

.. confval:: mds_max_readdir_entries
.. confval:: mds_max_readdir_bytes

Delegated inode numbers
-----------------------

Each client session holds a pool of preallocated inode numbers, up to
``mds_client_prealloc_inos``, so that creating a file does not need a new
allocation from the inode table. Part of that pool is handed to the client
itself: ``mds_client_delegate_inos_pct`` percent of it, topped up whenever fewer
than half remain. A kernel client mounted with async dirops (``nowsync``) uses
these delegated numbers to create a file locally and send the create to the MDS
without waiting for the reply; it may start writing file data under the new
inode number before the MDS has seen the create. The percentage is converted to
a fraction with integer division, so any value from 51 to 100 delegates the
whole pool, 34 to 50 delegates half of it, and so on.

Because of that early write, an inode number that is still delegated when the
session goes away may already back data objects the MDS knows nothing about. By
default the MDS deletes the first data object of each such number, one delete
per number. A session that is evicted, times out, is killed or is reclaimed
always gets this treatment.

A client that asks to close its session cleanly, with no request in flight, can
only have written under numbers it named in a create request. The MDS records
every delegated number a client names, before the request can be deferred,
forwarded to another rank or fail, and on such a clean close returns the numbers
the client never named to the inode table instead of deleting objects for them.
libcephfs and ceph-fuse never use delegated numbers, so their sessions now close
without any of these deletes. Set
``mds_session_close_free_unused_delegated_inos`` to false to purge every
delegated number, as before.

.. confval:: mds_session_close_free_unused_delegated_inos

The deletes that remain are issued in batches, at most
``mds_purge_inodes_max_ops`` in flight across all closing sessions. Sent all at
once, as they were before, the deletes of many sessions closing together fill
the OSD queues, and the journal writes of the rank wait behind them: on a single
rank with bluestore OSDs, 128 sessions closing at once delayed journal writes by
up to 30 seconds, long enough for a new client to time out mounting. With the
default of 512 the same burst kept journal writes under 80ms and took about as
long to drain. A value of 0 removes the limit.

.. confval:: mds_purge_inodes_max_ops

These deletes are separate from the purge queue, which removes the objects of
unlinked files and is throttled by ``mds_max_purge_ops`` and
``mds_max_purge_ops_per_pg`` (see :doc:`/cephfs/purge-queue`).

