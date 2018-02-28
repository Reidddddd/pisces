---
title: Filter in HBase
categories:
 - HBase
---

> Never did any man yet repent of having spoken too little, whereas many have been sorry they spok so much.

## Introduction

Filter plays an important role in querying data which can be used to reduce a mountain of unwanted data, especially in this data explosive era.  
There are plenty of filters provided by HBase to meet basic requirements, [available filters in HBase](http://hbase.apache.org/book.html#client.filter).  
Client can implement their own filter, but my personal experience is that it is not quite comprehensive to implement one, that's why i want to write it down after summary.

## Filter
Following are the core methods in `Filter`.
```java
public abstract class Filter {
	// Call between rows
	void reset() throws IOException;
	// Filter cell based on row key
	boolean filterRowKey(Cell firstRowCell) throws IOException;
	// Done with filtering
	boolean filterAllRemaining() throws IOException;
	// ReturnCode of filter result
	ReturnCode filterCell(final Cell c) throws IOException;
	// If transformation needed, not usual case
	Cell transformCell(final Cell v) throws IOException;
	// The results from filtering, this method provides a way to modify the results before returning to client side
	void filterRowCells(List<Cell> kvs) throws IOException;
	// The last chance to determine whether filter the entire row
	boolean hasFilterRow();
	// Called after hasFilterRow()
	boolean filterRow() throws IOException;
	// Fast filtering by returning a target cell to seek wanted position
	Cell getNextCellHint(final Cell currentCell) throws IOException;
	// Called at initialization
	boolean isFamilyEssential(byte[] name) throws IOException;

// ReturnCode from method filterCell()
enum ReturnCode {
	// Include the Cell
	INCLUDE,
	//Include the Cell and seek to the next column skipping older versions.
	INCLUDE_AND_NEXT_COL,
	// Skip this Cell
	SKIP,
	// Skip this column. Go to the next column in this row.
	NEXT_COL,
	/**
	* Seek to next row in current family. It may still pass a cell whose family is different but
	* row is the same as previous cell to {@link #filterCell(Cell)} , even if we get a NEXT_ROW
	* returned for previous cell.
	*/
	NEXT_ROW,
	// Seek to next key which is given as hint by the filter.
	SEEK_NEXT_USING_HINT,
	// Include KeyValue and done with row, seek to next. See NEXT_ROW.
	INCLUDE_AND_SEEK_NEXT_ROW,
    }
}
```

## Sequence
But it is still confusing, so following it is the call sequence.
![Filter call sequence](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/65d06d59c7c2afb5494e22b39ddcfb3a65c874e4/assets/images/filter.png)
1. `isFamilyEssential` is called first, but it does no filtering, it just helps to lock on the specified column family which may bring performance benefits by avoiding unnecessary column family scanning.
2. `filterRowKey` and `filterCell` can be regarded as entirety, first key then value, this philosophy is straightforward for a nosql system. `filterRowKey` is called to determine an entire row, while `filterCell` is called to determine a cell which return a returncode to indicate the next step. 
3. `filterAllRemaining` may be called between filter key and filter value, but it based on concrete implementation. Like `PageFilter` does filtering based on size of page, after a page is filled, `filterAllRemaining` is directly called to speed up filtering.
4. `getNextCellHint` is just another kind of `filterCell`, but it does further filtering based on targeting to wanted cell, and it will be called only after `filterAllRemaining` returns false and `filterCell` return `SEEK_NEXT_USING_HINT`.
5. `transformCell` is called to transform a cell into user wanted, but it is not often used.
6. `hasFilterRow`, `filterRow` and `filterRowCells` can be regarded as an entirety. If `hasFilterRow` returns false, those two left methods will not be called. `filterRowCells` is called at last to modify the pending return results list to client, it is not often used either. Then `filterRow` is called, the last chance to filter out entire row. Like `SingleColumnValueFilter` though found matched column, but didn't find matched value, and this row would be filtered out by `filterRow` method.
7. 1~6 should finish an entire row, and `reset` is called between rows to reset filter status

## Thoughts
- In server side code base, methods are called scatteredly, it can't be regard as a good looking framework.
- Considering combination of `Coprocessor` framework, it becomes more complicated, not to mention the procedure logic of scan/get.
