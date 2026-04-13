#### columns.tsx

```javascript
// Import ColumnDef and FilterFn types from TanStack React Table
import { type ColumnDef, type FilterFn } from "@tanstack/react-table";

// ✅ Added: Reusable text-based filter for column-specific searching
const textColumnFilter: FilterFn<z.infer<typeof schema>> = (
  row,
  columnId,
  filterValue,
) => {
  const value = String(row.getValue(columnId) ?? "").toLowerCase();
  const keyword = String(filterValue ?? "").toLowerCase().trim();

  // If no keyword is provided, do not filter the rows
  if (!keyword) return true;

  // Perform a case-insensitive partial match
  return value.includes(keyword);
};


// Enable column-specific search by adding `enableSearch` to the header
// and applying the reusable `textColumnFilter` to `filterFn`

// Before
{
  accessorKey: "name",
  header: "氏名",
},

// After
{
  accessorKey: "name",
  header: ({ column }) => (
    <SortableHeader
      column={column}
      title="氏名"
      enableSearch // Enables the search input for this column
    />
  ),
  filterFn: textColumnFilter, // ✅ Added: Apply the reusable text filter to the "name" column
},

```

#### 変更内容まとめ
- @tanstack/react-table から ColumnDef と FilterFn をインポート。
- 再利用可能なテキスト検索用フィルター textColumnFilter を実装。
- SortableHeader に enableSearch プロパティを追加し、列ごとの検索を可能に。
- 各カラム定義に filterFn を設定し、部分一致・大文字小文字を区別しない検索を実現。
- z.infer<typeof schema> を使用し、型安全性を向上。
- 保守性・再利用性を高め、ユーザー体験を向上。```

