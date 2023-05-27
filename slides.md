# Postgres Hacking: Exec Nodes

by Paul Jungwirth

Illuminated Computing

May 2023

Notes:

- Thank you, I'm Paul Jungwirth.
- I do independent consulting & development through my company Illuminated Computing.
- When I'm lucky I get to do Postgres work.
- I've been a minor contributor since 2016.
- I've been working on adding SQL:2011 temporal support for the last few years.

- As a newish contributor, I've often wished for a comprehensive guide to hacking on Postgres.
- There are some great articles and talks out there already,
  but one place I've never found much help is on the executor phase.
- So this is my contribution to helping others pick up Postgres hacking.



# Phases

- Parsing
- Analysis
- Rewriting
- Planning & Optimizing
- Executing

Notes:

- Before I dive into the details, here is a map of the territory.
- To execute a query, Postgres goes through several phases.
- It has to parse your SQL,
  Make sure it makes sense,
  Possibly expand views and apply rewrite rules,
  Build possible plan trees,
  Cost them and choose the best one,
  and finally run it!
- I want to give the talk I wish I had when I was working temporal stuff.
- There are lots of talks about writing custom C functions for Postgres,
  and lots about the parsing and analysis phases,
  but I could find very little about the execution phase---
  which is where the real work is supposed to happen.
- So I'm going to rush a bit through the early phases
  so I can talk more about executor nodes, tuple table slots, etc.
- This talk will probably be a bit desultory,
  but I hope following the pipeline structure will help a bit.
- I want to also use my temporal work as a running example.

- Of course PGCon is like the riskiest place to give a talk like this.
  - Here I am a Postgres dilettante you are all the real experts.
  - So I'll try to leave some time at the end for you to correct my mistakes.



# For Example

<pre class="code-wrapper">
<code class="text hljs plaintext" data-noescape><span class="fragment highlight-current-green">ALTER TABLE ADD PERIOD valid_at (valid_from, valid_til)</span>

<span class="fragment highlight-current-green">PRIMARY KEY/UNIQUE (id, valid_at WITHOUT OVERLAPS)</span>

<span class="fragment highlight-current-green">UPDATE/DELETE FROM t
  FOR PORTION OF valid_at
  FROM '2020-01-01' TO '2030-01-01'</span>

<span class="fragment highlight-current-green">FOREIGN KEY (id, PERIOD valid_at)
  REFERENCES parent (id, PERIOD valid_at)</span></code>
</pre>

Notes:

- Hopefully my temporal work will help make things concrete and motivated.
- There are four parts:
  - adding a PERIOD (which is a bit like a range but not a true column).
  - adding a temporal primary key or unique constraint (which includes a PERIOD or (in Postgres) a range column).
  - running a temporal UPDATE/DELETE, which targets just a span of time and leaves the other times untouched.
  - adding a temporal foreign key.
- Mostly I'll talk about `FOR PORTION OF`, because that's the most "interesting" command here.
  - Basically SELECT and DML commands go through the full query pipeline.
  - DDL are "utility commands" and get more direct treatment.
    - They are still parsed & analyzed ...
      but then we don't build up a plan or a tree of executor nodes.
      We just do it.



# Why

```text
On Tue, Nov 14, 2017 at 9:43 AM Tom Lane <tgl@sss.pgh.pa.us> wrote:
>
> Robert is correct that putting this into the parser
> is completely the wrong thing.
> If you do that, then for example views using the features
> will reverse-list in the rewritten form,
> which we Do Not Want,
> even if the rewritten form is completely valid SQL
> (is it?).
> 
> . . .
> 
>       regards, tom lane
```

Notes:

- Why it matters:
  - I've noticed several patches being rejected because they do their work in the wrong phase.
    - There was a really cool temporal patch a few years ago that did nearly everything in the analysis phase. This quote is from that patch.
    - Early versions of the MERGE command patch were rejected with similar feedback.
      - Robert Haas has an excellent post on that patch from 2019 giving specific reasons why you should do things in the right phase,
        as well as links to past feedback with yet more reasons.
        Robert and Tom's posts are both in the References at the end of my talk.
        I've been trying to find them again for years actually, but I could never get Google to give me what I wanted.
        Finally for this talk I downloaded all the mbox files and searched them locally. (Sorry archive team!)
        Now you can check them out yourselves!

  - Most commonly patches do too much upfront in analysis, but this has some problems:
    - When you describe a view, you won't get back the SQL you typed in.
    - The output from EXPLAIN will look funny.
      - Actually my own patch is maybe wrong here:
        FOR PORTION OF adds a qual (a WHERE condition basically)
        restricting it to rows that match the targeted timeframe.
        I decided that showing that in EXPLAIN seeemed helpful,
        although technically it's not something the user typed.
        It may not always be appropriate though, depending on the feature.

  - At one point `FOR PORTION OF` was requiring a temporal PK (which was not really correct),
    and that meant it didn't work against an updateable view,
    even though the view included the range column referenced by FOR PORTION OF.
    Updateable views don't get handled until the rewrite phase.

  - pg_dump??



# tcop/postgres.c
<!-- .slide: style="font-size: 80%" -->

```text [|1-2|1,3-4|1,3,5-7|1,8|1,9-12|1,13-17]
exec_simple_query
  pg_parse_query
  pg_analyze_and_rewrite
   parse_analyze
   pg_rewrite_query
     QueryRewrite
       RewriteQuery
  pg_plan_queries
  PortalDefineQuery
  PortalStart
    ExecutorStart
      ExecInitModifyTable
  PortalRun
    FillPortalStore
    PortalRunSelect
      ExecutorRun
        ExecModifyTable
```

Notes:

- Here are some of the major functions Postgres on its journey through the pipeline.
- There's parsing (highlight), analysis (highlight), rewriting (highlight).
  - The names here are kind of funny to me.
- planning (highlight)
- setting up the executor nodes (highlight)
- processing the executor nodes (highlight)



# Parsing

```c
/* src/backend/parser/gram.y */

for_portion_of_clause:
    FOR PORTION OF ColId FROM a_expr TO a_expr
      {
        ForPortionOfClause *n = makeNode(ForPortionOfClause);
        n->range_name = $4;
        n->range_name_location = @4;
        n->target_start = $6;
        n->target_end = $8;
        $$ = n;
      }
    | /*EMPTY*/         { $$ = NULL; }
  ;
```

Notes:

- Parsing & analysis are often grouped together.
- Postgres uses lex & yacc (or I should say flex & bison) to tokenize then parse your input, respectively.
- The big bison file is `gram.y`.
  - This defines the SQL grammar.
  - Here is a quote from it for the FOR PORTION OF clause.
  - When bishop sees the symbols in the rule, it runs the C code below.
- You can see a call to makeNode.
  - Parsing contructs a big parse tree made of these nodes.
  - Nodes are what flow through the pipeline.
  - We start with parse nodes,
    use those to make plan nodes, 
    and finally create exec nodes.



# Parse Nodes

```c
/* src/include/nodes/parsenodes.h */
/*
 * ForPortionOfClause
 *      representation of FOR PORTION OF <period-name>
 *                        FROM <t1> TO <t2>
 */
typedef struct ForPortionOfClause
{
    NodeTag     type;
    char       *range_name;
    int         range_name_location;
    Node       *target_start;
    Node       *target_end;
} ForPortionOfClause;
```

Notes:

- Here is a new node type: a ForPortionOfClause.
- There are big high-level nodes like SelectStmt,
- down through clauses like GROUP BY or OVER,
- and expressions made of operators, functions, literals, column references, etc.

- There are functions to make a node (i.e. allocate memory for it),
  to copy a node, to print a node, to read a node back from what was printed.
- Until recently you had to write all these each time you added a node,
  but now there is some clever codegen that does it for you.

- You will also see a lot of List members.
  - A List is also a Node, but it's a node that has a bunch of other nodes.
  - There are functions to build them, iterate over them, append to them.
  - When people say the Postgres is lispy, this is part of what they mean.

- Nodes & Lists are well-covered by other talks, so if you want to know more check the references at the end.



# Memory Contexts

```text
TopMemoryContext
PostmasterContext
CacheMemoryContext
MessageContext
TopTransactionContext
CurTransactionContext
PortalContext
ErrorContext
and more!
```

Notes:

So all these Nodes are getting allocated, right?
This is C isn't it?
Where do you think we free them?
We don't!
Postgres has an arena-based memory system.
Arenas are nested and have lifetimes from very broad to more specific:
For instance for the transaction, the current query, the function being called, the current row, etc.
When a context ends, everything in it gets freed.
To someone like me who writes a lot of Ruby and Python, this is awesome!
Just the pleasure of writing C with totally insoucient memory management makes me want to do more Postgres.



# Analysis

```c [|4|6-8|9-12|14]
static Query *
transformUpdateStmt(ParseState *pstate, UpdateStmt *stmt)
{
    Query      *qry = makeNode(Query);

    qry->resultRelation = setTargetTable(pstate, stmt->relation,
                                         stmt->relation->inh,
                                         true, ACL_UPDATE);
    if (stmt->forPortionOf)
        qry->forPortionOf = transformForPortionOfClause(
                pstate, qry->resultRelation,
                stmt->forPortionOf, true);

    transformFromClause(pstate, stmt->fromClause);
```

Notes:

- The bison file gives you a tree of what was typed,
  but it hasn't done any work yet to look up column names,
  validate your command, check permissions, etc.
- That's what the analysis phase does.
  - Here we can consult the database schema to find oids, types, etc.
- It will transform your parse nodetree (here an UpdateStmt) into a Query nodetree.
  You can see the top-level result node will be Query (highlight).
  So we're already starting to build plan nodes.
  The parse nodes should just capture what the user typed, as plainly as possible.
- So the first thing we're doing is looking up the table to update (highlight).
- Next if there was a `FOR PORTION OF` we translate that into plan nodes (highlight).
- Lots of the analysis work happens in functions named `transformThis` and `transformThat`,
  like here we process a `FROM` if there was one.



# syscache

```c [|1-3|4|6|7-9|14|10-11|13]
HeapTuple perTuple = SearchSysCache2(PERIODNAME,
                                     ObjectIdGetDatum(relid),
                                     PointerGetDatum(range_name));
if (HeapTupleIsValid(perTuple))
{
    Form_pg_period per = (Form_pg_period) GETSTRUCT(perTuple);
    Oid rngtypid    = per->perrngtype;
    int start_attno = per->perstart;
    int end_attno   = per->perend;
    Type rngtype    = typeidType(per->perrngtype);
    char *range_type_name = typeTypeName(rngtype);
    . . .
    ReleaseSysCache(rngtype);
    ReleaseSysCache(perTuple);
}
```

Notes:

- Syscache lets you look things up in the system catalog tables.
- We call it a lot in analysis,
  so this seems like a good place to cover it.
- Here `FOR PORTION OF` is looking up the period name.
- (transition to highlight that part)
- You see we're calling `SearchSysCache2`.
  This is defined in `utils/cache/syscache.c`.
  We give it the cache name, here `PERIODNAME` (highligh). That determines the table to query, the index the use, and how big the cache should be.
  This is search *2* because we search based on two attributes. Here the relation id (i.e. the table), and the range name. So those are the other arguments. (highlight)
- We don't necessarily get a result (highlight), so we have to check if we got something,
  and maybe report an error if not.
- Then we cast the `HeapTuple` to a struct matching the `pg_period` table.
- For simple types like ints and oids you can just pull them off the struct (highlight).
- When you're done you call `ReleaseSysCache` (highlight).
  - This isn't freeing memory but decrements a reference count
    so Postgres knows when to unlock the cache entry.
    If you don't call this it will be locked until the end of the transaction.
    If I recall correctly Postgres will scold you if you forget to do this.

- For fancier types there are helper functions you can use (highlight).
- Each PERIOD knows the range type used behind the scenes to implement various operations,
  so we look that up too.
- For the range type we have a nice helper function, typeidType.
  This is just defined with the parser code.
  A `Type` is just a typedef'd `HeapTuple`, so we have to free that the same way.
- Another helpful function here is `typeTypeName`,
  which gives us a C-string for the type.
- If you're lucky there will be a helpful function that calls `SearchSysCache` for you,
  so don't forget to check for that.
- Since it's really a HeapTuple, we free it the same way (highlight).



# lsyscache

```c
char *get_periodname(Oid periodid, bool missing_ok) {
 HeapTuple tp = SearchSysCache1(PERIODOID,
                                ObjectIdGetDatum(periodid));
 if (HeapTupleIsValid(tp)) {
   Form_pg_period period_tup = (Form_pg_period) GETSTRUCT(tp);
   char *result = pstrdup(NameStr(period_tup->pername)); 
   ReleaseSysCache(tp);
   return result;
 }
 
 if (!missing_ok)
   elog(ERROR, "cache lookup failed for period %d", periodid);
 return NULL;
}
```

Notes:

- Speaking of caches, there is also lsyscache.
- I don't know what the L stands for.
- It has a bunch of common helper routines on top of the syscache.
- They call `ReleaseSysCache` for you so they are convenient.
- It's common to call these during analysis, but really they're called anywhere.
- Here is a helper function I added.
- This is used by the foreign key triggers.



# typcache

```c [|2-4|5-7]
RangeType *r = DatumGetRangeTypeP(src->fp_targetRange);
TypeCacheEntry *typcache =
    lookup_type_cache(RangeTypeGetOid(r),
                      TYPECACHE_RANGE_INFO);
dst->fp_targetRange = datumCopy(src->fp_targetRange,
                                typcache->typbyval,
                                typcache->typlen);
```

Notes:

- And then one more cache you will surely use is the typcache.
- Types and operators are so important they have their own cache and helper functions.
  Not only do we need them to do almost anything,
  their info tends to be spread across a lot of tables, not just `pg_type`,
  so it would be expensive to look up all this stuff whenever we need it.
- This is all defined in `utils/cache/typcache.c`.
- Postgres assumes that types don't change very often.
  I mean int is never going out of style, right?
  So a `TypeCacheEntry` is never freed.
  This saves a lot of work---both for Postgres and for you.
  I guess you don't want to operationally redefine types, because you'd be leaking memory.
- The typcache has some functions for domains and enums, but the only really essential function is `lookup_type_cache` (highlight).
  - A `TypeCacheEntry` is a big struct with lots of info, and you might not want all of it, so you can use constants like `TYPECACHE_RANGE_INFO` to say what you care about.
  - Here (highlight) we are using the type's size and whether it's pass-by-value to make a copy of a range.



# RangeTblEntry

```
TODO: show getting this off pstate first to motivate it.
```

```c
typedef struct RangeTblEntry
{
    NodeTag     type;

    RTEKind     rtekind;        /* see above */
    Oid         relid;          /* OID of the relation */
    char        relkind;        /* relation kind (see pg_class.relkind) */
    int         rellockmode;    /* lock level that query requires on the rel */
    Index       perminfoindex;
    Query      *subquery;       /* the sub-query */
    /* . . . */
```

Notes:

- In the Postgres source you'll find many references to RangeTblEntries (or RTEs).
- A range table is:
  1. not a database table (i.e. not a relation).
  2. has nothing to do with range types.
- It's a table as in a list of structs.
- Each struct is a quote-unquote range: which is basically a relation:
  - either a true persisted table,
  - or the result of a subquery or join,
  - or some other more exotic thing.
- So when your query joins a bunch of tables or subqueries, each FROM entry is a RangeTbl*Entry*: an entry in the range table.




# Rewriting

- `VIEW`s
- `RULE`s
- `Query` -> List (of `Query`)

Notes:

I don't want to spend time on this, but this is where we expand VIEWs and apply RULEs.
The main functions take a Query node and return a List of zero or more Query nodes.
In the really easy cases we're just wrapping the passed-in Query node.



# Rewriting

```c [|1|8-12]
foreach(lc, parsetree->forPortionOf->rangeSet)
{
    TargetEntry *tle = (TargetEntry *) lfirst(lc);
    TargetEntry *view_tle;

    if (tle->resjunk) continue;

    view_tle = get_tle_by_resno(view_targetlist, tle->resno);
    if (view_tle != NULL &&
            !view_tle->resjunk &&
            IsA(view_tle->expr, Var))
        tle->resno = ((Var *) view_tle->expr)->varattno;
    else
        elog(ERROR, "attribute number %d not found in view targetlist", tle->resno);
}
```

Notes:

- I did have to little bit here to make sure we could use `FOR PORTION OF` against an updatable view.
- If your feature should work against a view, maybe you need to do something here.
- This might also be a worthwhile place for doing things you aren't supposed to do in analysis.
- For example with UPDATE FOR PORTION OF, we are implicitly setting the `PERIOD` start/end columns.
  - The means we need `TargetEntry` nodes for those columns.
  - An UPDATE has a list of TargetEntry nodes.
    - Often these are called TLEs for "Target List Entry".
    - Each of these represents a column to update.
      - They have the attribute number and some other info.
    - In other contexts they might be columns to SELECT, etc.
    - Our `forPortionOfExpr` has a list of TLEs for the implicitly-set columns, separate from the ones the user sets explicitly. (highlight)
  - In you have an updatable view, we need to convert our TLE from the view's attno to the underlying table's.
- So I'm not saying much about rewriting, but TLEs are something you'll see elsewhere too, and this is a good excuse to mention them.



# Planning & Optimizing

- `Query` -> `PlannedStmt`

Notes:

- For each Query node the planner returns a PlannedStmt node.
- For each base relation the planner will generate several "Paths"
  then choose the best one: maybe a full-table scan, maybe an index scan, maybe a bitmap index scan.
- Also for each pair of joined relations the planner will generate Paths to implement the join.
  - One relation is the the outer and the other the inner.
  - It will consider a nested loop join, a hash join, etc.
- So all these paths get an estimated cost, and the planner chooses the best one.
- When I worked on multiranges I had to do some work to collect statistics about them and use those to make selectivity estimates.
- Okay that's all I know about the planner!



# Executor

```text [|1,9-10|2,11|2-8|11-13,16-18]
PortalStart
  ExecutorStart
    CreateExecutorState
    InitPlan
      ExecInitNode
        ...
        ExecInitModifyTable
        ...
PortalRun
  PortalRunSelect
    ExecutorRun
      ExecProcNode
        ExecModifyTable
```


Notes:

- Once we're ready to run the query, we pass our plan tree to the executor phase.
- This goes through a Portal (highlight), which is basically a door to pass result tuples from the backend to the client (or maybe directly to stdout).
  - Mostly this is how we implement cursors.
  - We can ignore it more or less.
- But the portal functions call ExecutorStart and ExecutorRun (highlight).
- ExecutorStart sets up another node tree.
  - `CreateExecutorState` creates an EState struct which has info about the overall execution.
  - Most plan nodes require some mutable state to execute, so each of them gets a sibling execState node.
  - ExecInitNode knows how to create the right execState for each kind of plan node.
    - Basically it's a big switch statement.
    - Then those functions it calls recursively call the right ExecInit function for their own children,
      and so on.
      Or maybe they call ExecInitNode again.
        - That's what ExecInitModifyTable does to set up its source of tuples to insert/update/delete.
- And then `ExecutorRun` (highlight) actually does the work.
- There are `ExecutorFinish` and `ExecutorEnd`
  as well as node-specific end functions, but I'm going to skip those.



# Executor

```text
typedef struct PlanState
{
    pg_node_attr(abstract)
    NodeTag     type;
    Plan       *plan;
    EState     *state;
    ExecProcNodeMtd ExecProcNode;
    ...
```

Notes:

- Here is the "abstract superclass" for our execState nodes.
- Each one has a type, just like any node (highlight)
- Each gets a reference to its plan node.
  - Whereas the execState is mutable, the plan node is stable for the whole executor phase.
- They all reference the top-level EState struct.
- They also each reference a function to process their kind of executor node.
  - Later we'll chase these function pointers to do all the work.
- There's a lot more stuff in this struct, but I'll skip it for now.



# Executor

```c
ExprState *exprState = ExecPrepareExpr(
        (Expr *) forPortionOf->targetRange, estate);
Datum targetRange = ExecEvalExpr(exprState, econtext, &isNull);
resultRelInfo->ri_forPortionOf->fp_targetRange = targetRange;
```

Notes:

- Here is a bit of what we do to init our ForPortionOfState.
- We need to evaluate the FROM and TO parameters and turn those into a range.
  - These are allowed to be functions like `NOW()` or arithmetic like `NOW() + INTERVAL '1 day'`.
- Then as we update each row we can use range functions to see if the old row covered any time that should not be updated.
  - The SQL:2011 standard says when we update the original row we should force the start/end bounds to the period targeted by FOR PORTION OF . . .
  - and then implicitly INSERT up to two new rows to preserve the "leftovers" the UPDATE didn't target.
- The TO & FROM can't change row-by-row, so the Init step is a good place to build our range.
- We already constructed the expression tree in the analysis phase.
- Now we evaluate it.



# Executor

```c
/* Initialize slot for the existing tuple */

resultRelInfo->ri_forPortionOf->fp_Existing =
    table_slot_create(resultRelInfo->ri_RelationDesc,
                      &mtstate->ps.state->es_tupleTable);

/* Create the tuple slots for INSERTing the leftovers */

resultRelInfo->ri_forPortionOf->fp_Leftover1 =
    ExecInitExtraTupleSlot(mtstate->ps.state, tupDesc,
                           &TTSOpsVirtual);
resultRelInfo->ri_forPortionOf->fp_Leftover2 =
    ExecInitExtraTupleSlot(mtstate->ps.state, tupDesc,
                           &TTSOpsVirtual);

```

Notes:

- Another thing we do in the Init step is create some TupleTableSlots.
- A TupleTableSlot is a place to hold a tuple.
  - We saw this briefly before.
- We need a place for three tuples:
  - one for the row we just updated,
  - two for the "leftover" rows we want to insert:
    - one for the time before the target range, one after.



# Executor

```text
ExecutorRun
  ExecutePlan
    for (;;) {
      slot = node->ExecProcNode(node);
      if (TupIsNull(slot))
          break;
    }
}
  
```

Notes:

- Okay so that's `ExecutorStart`, which gets us ready to call `ExecutorRun`.
- This calls ExecutePlan,
- And that calls ExecProcNode, which is the partner of ExecInitNode, which we saw above.
  - ExecProcNode is a tiny inline function that calls the node's function pointer.
  - We call it over & over, and each time it processes one row and returns the result.
  - The result is a `TupleTableSlot` (highlight): like a just saw!
  - If the node has children, somewhere in its proc function it calls the proc function for those children too.
    - For instance maybe you have a node to do a NestedLoop, which you've probably seen in `EXPLAIN` output.
      - It needs the rows from its inner & outer relations, so it calls the proc function for those nodes.
    - So we've got this whole tree of proc functions passing around tuples.
  - Incidentally using function pointers makes it easy to inject instrumentation.
    - I haven't checked but I'm betting that's how `EXPLAIN ANALYZE` works.



# Executor

```text
for (;;) {
  context.planSlot = ExecProcNode(subplanstate);
  if (TupIsNull(context.planSlot)) break;

  switch (operation) {
    case CMD_UPDATE:
      slot = ExecUpdate(...);
  }
}
```

Notes:

- Here are some bits from ExecModifyTable.
  - We use this same node for INSERT, UPDATE, DELETE, and MERGE.
- You can see we're calling the proc function of our subplan to get rows (highlight),
- and then we call a function for the specific operation we want (highlight).
- ExecUpdate will call ExecUpdatePrologue, ExecUpdateAct, and ExecUpdateEpilogue.



# Executor

```c
RangeType *oldRange = slot_getattr(oldtupleSlot,
        forPortionOf->rangeVar->varattno, &isNull);
RangeType *targetRange = DatumGetRangeTypeP(
        resultRelInfo->ri_forPortionOf->fp_targetRange);

range_leftover_internal(typcache,
        oldRangeType, targetRangeType,
        &leftoverRangeType1, &leftoverRangeType2);
```

Notes:

- If the UPDATE (or DELETE) had a FOR PORTION OF clause,
  then we call an ExecForPortionOf function.
- It gets the row's old range value.
- It gets the target range that we set up above.
- It finds out if there are any leftovers to the left or right.
  - There are more `RangeType` pointers.



# Executor

```c
if (!RangeIsEmpty(leftoverRangeType1))
{
  MinimalTuple oldtuple = ExecFetchSlotMinimalTuple(
          oldtupleSlot, NULL);
  ExecForceStoreMinimalTuple(oldtuple, leftoverTuple1, false);

  set_leftover_tuple_bounds(leftoverTuple1, forPortionOf,
                            typcache, leftoverRangeType1);
  ExecMaterializeSlot(leftoverTuple1);

  ExecInsert(context, resultRelInfo, leftoverTuple1,
             node->canSetTag, NULL, NULL);
}
```

Notes:

- And now if either of those leftover ranges are non-empty,
- it takes the pre-update values from oldtuple,
  copies them into our leftover TupleTableSlot,
  changes the start/end values,
  and does an Insert with it.
- This is for leftoverRangeType1 but there is similar code for the other side.
  In fact I should probably move this into a little helper function.



# Tuple Table Slots

```c
typedef struct TupleTableSlot
{
    NodeTag     type;
    uint16      tts_flags;      /* Boolean states */
    AttrNumber  tts_nvalid;     /* # of valid values in tts_values */
    const TupleTableSlotOps *const tts_ops; /* implementation of slot */
    TupleDesc   tts_tupleDescriptor;    /* slot's tuple descriptor */
    Datum      *tts_values;     /* current per-attribute values */
    bool       *tts_isnull;     /* current per-attribute isnull flags */
    MemoryContext tts_mcxt;     /* slot itself is in this context */
    ItemPointerData tts_tid;    /* stored tuple's tid */
    Oid         tts_tableOid;   /* table oid of tuple */
} TupleTableSlot;
```

Notes:

- So what are these TupleTableSlots anyway?
- If you're working in the executor you'll probably need to use one.
- I couldn't find any talk or article talking about these things.
- The place to look is `executor/tuptable.h`.
- Of course a tuple is more-or-less a row.
- A "tuple table" is how the executor deals with processing tuples.
  - It's a table like the Range Table is a table: a list of structs.
  - Each tuple is kept in a TupleTableSlot.
  - Guess what? This is a node too (highlight)! But you don't really mix it with other nodes.

- You can see we've got a list of `Datum`s and null flags (highlight).
  - Every other intro to Postgres talks about Datums, so I've kind of shied away from giving you lots of details here.
  - But basically a Datum is a single value: maybe of a row attribute, or an evaluated expression, or whatever.
  - It doesn't matter what type the data is.
  - There are functions and macros to cover them to/from concrete types, e.g. `DatumGetInt64` or `Int64GetDatum`.
    - If the type is something small enough to be pass-by-value, this should be just a cast.
    - Otherwise the `Datum` is a pointer and might be TOASTed, so `DatumGetFoo` will also de-toast it for you.
    - If you have written any custom C functions, you've worked with `Datum`s before.
    - Lots of links in my references have more details.

- These two arrays basically *are* the tuple, but the TupleTableSlot lets us easily pass around other important information *about* the tuple.
  - For example the tuple descriptor (highlight), which tells us how many attributes there are and what their types are---or actually the whole `pg_attribute` record if possible.

- Now there are four kinds of TupleTableSlot:
  - You see this `tts_ops` field (highlight).
    - That points to an 'Ops struct which is a big list of function pointers.
    - So this is our dynamic dispatch and basically determines what kind of TupleTableSlot we've got.



# Tuple Table Slots
## TTSOpsBufferHeapTuple

```text
ExecStoreBufferHeapTuple
```

Notes:

- Suppose you're a SeqScan Node.
  - Your Init function sets up a TupleTableSlot to hold the rows you're going to return.
  - Maybe you have another one if you have to do any projections.
    - You know what I mean by projection? Like concat `first_name` and `last_name`.
  - Then in your proc node you're asking an Access Method for the tuples from the table or index,
    and you call `ExecStoreBufferHeapTuple` to put it into your TupleTableSlot.
    - So that puts a pin in the buffer page (so it doesn't go away),
      then it stores a pointer to the `HeapTuple` struct.



# Tuple Table Slots
## TupleTableSlot: TTSOpsHeapTuple

Notes:

- A HeapTuple slot points to palloc'd memory instead of a buffer page.
- So instead of releasing a pin when we're done, we want to free the memory.
- For both of these TupleTableSlots, you can pull the data out into the Datum and isnull arrays and work with it.
  - If the Datum is pass-by-reference, it just points into the tuple data.



# Tuple Table Slots
## TTSOpsMinimalTuple

```c
/* Get the range of the existing pre-UPDATE/DELETE tuple */

if (!table_tuple_fetch_row_version(
      resultRelInfo->ri_RelationDesc, tupleid,
      SnapshotAny, oldtupleSlot))
  elog(ERROR, "failed to fetch tuple for FOR PORTION OF");

oldRange = slot_getattr(oldtupleSlot,
                        forPortionOf->rangeVar->varattno,
                        &isNull);
```

Notes:

- A minimal tuple is like a palloc'ed tuple, but it has no system columns.
- We use this to hold the old version of the row being UPDATEd or DELETEd.



# Tuple Table Slots
## `TTSOpsVirtual`

```c [|1-3|5-7|9-10]
MinimalTuple oldtuple = ExecFetchSlotMinimalTuple(
    oldtupleSlot, NULL);
ExecForceStoreMinimalTuple(oldtuple, leftoverTuple1, false);

set_leftover_tuple_bounds(leftoverTuple1, forPortionOf,
                          typcache, leftoverRangeType1);
ExecMaterializeSlot(leftoverTuple1);

ExecInsert(context, resultRelInfo, leftoverTuple1,
           node->canSetTag, NULL, NULL);
```

Notes:

- A virtual TupleTableSlot doesn't own its own tuple memory.
- The Datum and isnull arrays are the authoritative data.
- Using these can prevent copying between plan nodes.
- We use these to store the new rows for "leftovers" untouched by the `FOR PORTION OF`.
- Here (highlight) we take the old version of the row and copy it into our virtual tuple.
- Then (highlight) we compute the new start/end times
  and mark the tuple as "ready".
  - `ExecMaterializeSlot` basically marks the tuple as "ready".
  - TODO: no! For other tuple types is copies the data into the Datum and isnull arrays.
- And then we can ask this tuple to be inserted.



# TODO

ExecInitNullTupleSlot
ExecInitResultTupleSlotTL
ExecInitExtraTupleSlot
MakeSingleTupleTableSlot
MakeTupleTableSlot



# References
<!-- .slide: style="font-size: 50%; text-align: left" class="bibliography" -->

1. Selena Deckelmann, *So, you want to a developer*, 2011. https://wiki.postgresql.org/wiki/So,_you_want_to_be_a_developer%3F

1. Laetitia Avrot, *Demystifying Contributing to PostgreSQL*, 2018. https://www.slideshare.net/LtitiaAvrot/demystifying-contributing-to-postgresql

1. Neil Conway and Gavin Sherry, *Introduction to Hacking PostgreSQL*, 2007. http://www.neilconway.org/talks/hacking/hack_slides.pdf and https://www.cse.iitb.ac.in/infolab/Data/Courses/CS631/PostgreSQL-Resources/hacking_intro.pdf

1. Greg Smith, *Exposing PostgreSQL Internals with User-Defined Functions*, 2010. https://www.pgcon.org/2010/schedule/attachments/142_HackingWithUDFs.pdf

1. Hironobu Suzuki, *The Internals of PostgreSQL*, 2012. http://www.interdb.jp/pg/

1. Egor Rogov, *Indexes in PostgreSQL*, 2019. https://habr.com/ru/companies/postgrespro/articles/441962/

1. Tom Lane, *Re: [HACKERS] [PROPOSAL] Temporal query processing with range types*, pgsql-hackers mailing list, 2017. https://www.postgresql.org/message-id/32265.1510681378@sss.pgh.pa.us

1. Robert Haas, *Re: MERGE SQL statement for PG12*, pgsql-hackers mailing list, 2019. https://www.postgresql.org/message-id/CA%2BTgmoZj8fyJGAFxs%3D8Or9LeNyKe_xtoSN_zTeCSgoLrUye%3D9Q%40mail.gmail.com
