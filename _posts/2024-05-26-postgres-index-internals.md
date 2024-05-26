---
layout: post
title: "PostgreSQL Index Internals"
subtitle: A quick overview of PostgreSQL index API
cover-img: assets/img/db-index.jpeg
tags: [databases]
---
In this post, I explain how PostgreSQL internally interacts with database indexes, and how you can develop a custom index.

# Indexes Overview
Indexes allow for quick retrieval of table entries. Without indexes, the database has to scan the entire table to find our queried information. However, with a good index, the database can find the entries relatively quickly.  
PostgreSQL supports several indexes, such as B-tree, Hash, GiST, SP-GiST, GIN, BRIN, and the bloom extension. However, the index access API allows for creating new indexes. In the next sections, I explain how PostgreSQL index API works.  

# PostgreSQL Indexes
PostgreSQL indexes are secondary indexes, which means they are physically separate from the tables. Indexes store their structures in standard-sized pages. This is a deliberate choice to allow indexes to use the same storage and buffer manager without needing to implement one themselves. It prevents lots of extra work.  
PostgreSQL offers an Index Access Method (IndexAM) interface for developers to write their indexes. The database doesn't care about the internal index implementation details until the index satisfies the **IndexAM** interface.

## PostgreSQL IndexAM Interface
The IndexAM interface is provided below. I explain the functions in more detail; the other parts are self-explanatory.
```c
typedef struct IndexAmRoutine
{
    NodeTag     type;

    /*
     * Total number of strategies (operators) by which we can traverse/search
     * this AM.  Zero if AM does not have a fixed set of strategy assignments.
     */
    uint16      amstrategies;
    /* total number of support functions that this AM uses */
    uint16      amsupport;
    /* opclass options support function number or 0 */
    uint16      amoptsprocnum;
    /* does AM support ORDER BY indexed column's value? */
    bool        amcanorder;
    /* does AM support ORDER BY result of an operator on indexed column? */
    bool        amcanorderbyop;
    /* does AM support backward scanning? */
    bool        amcanbackward;
    /* does AM support UNIQUE indexes? */
    bool        amcanunique;
    /* does AM support multi-column indexes? */
    bool        amcanmulticol;
    /* does AM require scans to have a constraint on the first index column? */
    bool        amoptionalkey;
    /* does AM handle ScalarArrayOpExpr quals? */
    bool        amsearcharray;
    /* does AM handle IS NULL/IS NOT NULL quals? */
    bool        amsearchnulls;
    /* can index storage data type differ from column data type? */
    bool        amstorage;
    /* can an index of this type be clustered on? */
    bool        amclusterable;
    /* does AM handle predicate locks? */
    bool        ampredlocks;
    /* does AM support parallel scan? */
    bool        amcanparallel;
    /* does AM support columns included with clause INCLUDE? */
    bool        amcaninclude;
    /* does AM use maintenance_work_mem? */
    bool        amusemaintenanceworkmem;
    /* does AM summarize tuples, with at least all tuples in the block
     * summarized in one summary */
    bool        amsummarizing;
    /* OR of parallel vacuum flags */
    uint8       amparallelvacuumoptions;
    /* type of data stored in index, or InvalidOid if variable */
    Oid         amkeytype;

    /* interface functions */
    ambuild_function ambuild;
    ambuildempty_function ambuildempty;
    aminsert_function aminsert;
    ambulkdelete_function ambulkdelete;
    amvacuumcleanup_function amvacuumcleanup;
    amcanreturn_function amcanreturn;   /* can be NULL */
    amcostestimate_function amcostestimate;
    amoptions_function amoptions;
    amproperty_function amproperty;     /* can be NULL */
    ambuildphasename_function ambuildphasename;   /* can be NULL */
    amvalidate_function amvalidate;
    amadjustmembers_function amadjustmembers; /* can be NULL */
    ambeginscan_function ambeginscan;
    amrescan_function amrescan;
    amgettuple_function amgettuple;     /* can be NULL */
    amgetbitmap_function amgetbitmap;   /* can be NULL */
    amendscan_function amendscan;
    ammarkpos_function ammarkpos;       /* can be NULL */
    amrestrpos_function amrestrpos;     /* can be NULL */

    /* interface functions to support parallel index scans */
    amestimateparallelscan_function amestimateparallelscan;    /* can be NULL */
    aminitparallelscan_function aminitparallelscan;    /* can be NULL */
    amparallelrescan_function amparallelrescan;    /* can be NULL */
} IndexAmRoutine;
```

## IndexAM Important Functions
* `IndexBuildResult* ambuild(Relation heap, Relation index, IndexInfo *indexInfo)`: This function gets the table (heap) and builds the index. It should store the built index in database pages. 
* `bool aminsert(Relation index, Datum *values, bool *isnull, ItemPointer heap_tid, Relation heap, IndexUniqueCheck checkUnique, IndexInfo *indexInfo)`: The index should find the correct index page based on the values. It then adds the new entry to the page (or adds a new page if the page doesn't have enough space). `heap_tid` is the id of the entry in the original table (PostgreSQL calls the original table heap).
* `IndexBulkDeleteResult* ambulkdelete(IndexVacuumInfo *info, IndexBulkDeleteResult *stats, IndexBulkDeleteCallback callback, void *callback_state)`: This function should iterate over the index pages and find the entries in those pages. It typically calls the [`PageIndexMultiDelete`](https://doxygen.postgresql.org/bufpage_8c.html#a6ada701d2748acf8d3fdd7e3e619f3b1) to delete the tuples from the index.
* `void amcostestimate(PlannerInfo *root, IndexPath *path, double loop_count, Cost *indexStartupCost, Cost *indexTotalCost, Selectivity *indexSelectivity, double *indexCorrelation, double *indexPages)`: This function estimates the cost of an index scan. This is necessary to allow the planner to decide on the best options and whether to use the index or not.
* `void amrescan(IndexScanDesc scan, ScanKey keys, int nkeys, ScanKey orderbys, int norderbys)`: intializes necessary stuff to get index elements using **amgettuple**.
* `void amendscan(IndexScanDesc scan)`: ends the index scan.
* `bool amgettuple(IndexScanDesc scan, ScanDirection dir)`: fetches the next tuple in the index scan. This function does the actual index search.

# Final Thoughts
I like how the index API allows developers to add new indexes to PostgreSQL. There are many extensions with new indexes for different use cases. I learned a lot by reading their source code, and how they implement the previous functions. [pgvector](https://github.com/pgvector/pgvector) is a good example of writing custom indexes to support vector similarity searches. The source code of their two main indexes is [here](https://github.com/pgvector/pgvector/blob/ff9b22977e3ef19866d23a54332c8717f258e8db/src/ivfflat.h) and [here](https://github.com/pgvector/pgvector/blob/ff9b22977e3ef19866d23a54332c8717f258e8db/src/hnsw.h).

# Further Reading
[PostgreSQL Docs](https://www.postgresql.org/docs/current/indexam.html)