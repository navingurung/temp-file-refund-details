# Sortable Header with Column Search -- Implementation Guide

## Overview

This document summarizes the enhancements made to the data table to
support **column-specific search** within the `SortableHeader` component
using **TanStack Table**. It also explains the reasoning behind each
change so the same implementation can be reused in future projects.

------------------------------------------------------------------------

## 1. Goals

-   Enable **per-column search** directly from the column header.
-   Maintain the existing **global search** functionality.
-   Provide a **reusable and scalable** filtering mechanism.
-   Improve **user experience (UX)** with a clear (✕) button.
-   Ensure clean and maintainable code for future reuse.

------------------------------------------------------------------------

## 2. Key Concepts

### 2.1 Global Filter vs Column Filter

  ------------------------------------------------------------------------
  Feature             Purpose                State Used
  ------------------- ---------------------- -----------------------------
  Global Search       Searches across        `globalFilter`
                      multiple columns       
                      (e.g., ID, Name,       
                      Nation)                

  Column Search       Searches within a      `columnFilters`
                      specific column        
  ------------------------------------------------------------------------

**Reason:** TanStack Table separates global and column filtering to
provide flexibility and better control over search behavior.

------------------------------------------------------------------------

## 3. Changes Implemented

### 3.1 Added `textColumnFilter` (Reusable Filter Logic)

**Location:** `columns.tsx`

``` ts
import { type FilterFn } from "@tanstack/react-table";

const textColumnFilter: FilterFn<any> = (row, columnId, filterValue) => {
  const value = String(row.getValue(columnId) ?? "").toLowerCase();
  const keyword = String(filterValue ?? "").toLowerCase().trim();

  if (!keyword) return true;

  return value.includes(keyword);
};
```

**Reason:** - Provides **case-insensitive partial matching**. - Reusable
across multiple text-based columns. - Ensures consistent filtering
behavior.

------------------------------------------------------------------------

### 3.2 Applied Column Filter to Searchable Columns

``` ts
{
  accessorKey: "id",
  header: ({ column }) => (
    <SortableHeader column={column} title="ID" enableSearch />
  ),
  enableHiding: false,
  filterFn: textColumnFilter,
},
{
  accessorKey: "shopName",
  header: ({ column }) => (
    <SortableHeader column={column} title="店舗名" enableSearch />
  ),
  filterFn: textColumnFilter,
},
{
  accessorKey: "name",
  header: ({ column }) => (
    <SortableHeader column={column} title="氏名" enableSearch />
  ),
  filterFn: textColumnFilter,
},
{
  accessorKey: "nation",
  header: ({ column }) => (
    <SortableHeader column={column} title="国籍" enableSearch />
  ),
  filterFn: textColumnFilter,
}
```

**Reason:** - Enables column-specific searching. - Keeps sorting and
filtering behavior consistent.

------------------------------------------------------------------------

### 3.3 Updated `SortableHeader` to Include Search Input

``` tsx
<Input
  value={(column.getFilterValue() as string) ?? ""}
  onChange={(e) => column.setFilterValue(e.target.value)}
  placeholder={`${title}を検索`}
  className="h-8 pl-7 pr-7"
/>
```

**Reason:** - Connects the UI input to TanStack's `columnFilters`
state. - Allows users to filter data directly from the column header.

------------------------------------------------------------------------

### 3.4 Added Clear (✕) Button to Reset Input

``` tsx
{(column.getFilterValue() as string) && (
  <button
    type="button"
    onClick={(e) => {
      e.stopPropagation();
      column.setFilterValue("");
    }}
    className="text-muted-foreground hover:text-foreground absolute top-1/2 right-2 -translate-y-1/2 cursor-pointer"
    aria-label="検索をクリア"
  >
    <IconX className="h-3.5 w-3.5" />
  </button>
)}
```

**Reason:** - Improves **usability** by allowing quick reset of the
filter. - Provides **visual feedback** when a filter is active. - Aligns
with modern UI/UX standards.

------------------------------------------------------------------------

### 3.5 Enhanced Reset (`リセット`) Behavior

``` tsx
<DropdownMenuItem
  onClick={() => {
    column.clearSorting();
    if (enableSearch) {
      column.setFilterValue("");
    }
  }}
>
  <IconRotate className="h-4 w-4" />
  リセット
</DropdownMenuItem>
```

**Reason:** - Ensures both **sorting** and **filtering** are reset
together. - Provides a consistent and intuitive user experience.

------------------------------------------------------------------------

### 3.6 Ensured Proper Table State Configuration

``` ts
const [globalFilter, setGlobalFilter] = React.useState("");
const [columnFilters, setColumnFilters] = React.useState<ColumnFiltersState>([]);

const table = useReactTable({
  data,
  columns,
  state: {
    sorting,
    columnVisibility,
    rowSelection,
    columnFilters,
    globalFilter,
    pagination,
  },
  onColumnFiltersChange: setColumnFilters,
  onGlobalFilterChange: setGlobalFilter,
  getFilteredRowModel: getFilteredRowModel(),
});
```

**Reason:** - `globalFilter` handles toolbar-wide search. -
`columnFilters` manages per-column filtering. - Prevents TypeScript
errors such as duplicate declarations.

------------------------------------------------------------------------

## 4. UX Improvements

  Enhancement                Benefit
  -------------------------- ------------------------------------
  Search icon                Indicates the purpose of the input
  Clear (✕) button           Quick reset of filters
  Reset menu option          Clears both sorting and filtering
  Reusable filter function   Consistent behavior across columns
  Wider dropdown (`w-48`)    Better layout for input and icons

------------------------------------------------------------------------

## 5. Final Architecture

    Toolbar Search  ---> globalFilter
    Header Search   ---> columnFilters
    Status Dropdown ---> columnFilters (custom filterFn)

------------------------------------------------------------------------

## 6. Reusability for Future Projects

To reuse this implementation:

1.  Copy `SortableHeader.tsx` with the search and clear functionality.
2.  Add `textColumnFilter` to `columns.tsx`.
3.  Apply `filterFn: textColumnFilter` to searchable columns.
4.  Ensure `columnFilters` and `globalFilter` are configured in
    `useReactTable`.
5.  Optionally adjust placeholders and labels.

------------------------------------------------------------------------

## 7. Summary

  Feature                     Status
  --------------------------- ----------------
  Global Search               ✅ Implemented
  Column Search               ✅ Implemented
  Reusable Filter Logic       ✅ Implemented
  Clear (✕) Button            ✅ Implemented
  Reset Sorting & Filtering   ✅ Implemented
  Reusable Architecture       ✅ Implemented

------------------------------------------------------------------------

## 8. Conclusion

This enhancement provides a **scalable, reusable, and user-friendly**
solution for integrating column-specific search into a TanStack Table.
The architecture cleanly separates responsibilities between UI
components and filtering logic, making it easy to replicate in future
dashboards or data-driven applications.

------------------------------------------------------------------------

*Prepared for future reuse and documentation purposes.*
