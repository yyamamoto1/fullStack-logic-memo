# ダッシュボード分析ツール ⭐⭐⭐

**難易度**: 上級
**推奨時間**: 25-35時間
**技術スタック**: React, TypeScript, Chart.js, D3.js, TanStack Table, Recharts

---

## 概要

大量のデータを可視化するダッシュボードツールを構築します。データ可視化ライブラリの使い方と大量データの効率的な処理方法を学びます。

---

## 学習目標

- [ ] Chart.js/Rechartsによるグラフ描画
- [ ] D3.jsによる高度な可視化
- [ ] TanStack Tableによる高機能テーブル
- [ ] 仮想スクロール（大量データ対応）
- [ ] データフィルタリング・集計
- [ ] CSV/PDFエクスポート

---

## 機能要件

### 必須機能

1. **ダッシュボード**
   - KPIカード表示
   - グラフウィジェット（折れ線、棒、円、エリア）
   - リアルタイムデータ更新
   - カスタマイズ可能なレイアウト

2. **データ可視化**
   - 時系列グラフ
   - 比較グラフ
   - ヒートマップ
   - ツリーマップ
   - 地図可視化

3. **データテーブル**
   - ソート・フィルター
   - 列のカスタマイズ
   - ページネーション/仮想スクロール
   - 行選択・一括操作

4. **エクスポート機能**
   - CSV出力
   - PDF出力
   - 画像出力（グラフ）

---

## 実装例

### グラフコンポーネント

```typescript
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
  ResponsiveContainer
} from 'recharts';

interface TimeSeriesData {
  date: string;
  value: number;
  previousValue?: number;
}

interface LineChartProps {
  data: TimeSeriesData[];
  title: string;
  yAxisLabel?: string;
  showComparison?: boolean;
}

export function TimeSeriesChart({
  data,
  title,
  yAxisLabel,
  showComparison = false
}: LineChartProps) {
  return (
    <div className="bg-white rounded-lg shadow p-4">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis
            dataKey="date"
            tickFormatter={(value) => formatDate(value)}
          />
          <YAxis
            label={{ value: yAxisLabel, angle: -90, position: 'insideLeft' }}
            tickFormatter={(value) => formatNumber(value)}
          />
          <Tooltip
            formatter={(value: number) => formatNumber(value)}
            labelFormatter={(label) => formatDate(label)}
          />
          <Legend />
          <Line
            type="monotone"
            dataKey="value"
            stroke="#3B82F6"
            strokeWidth={2}
            dot={false}
            name="今期"
          />
          {showComparison && (
            <Line
              type="monotone"
              dataKey="previousValue"
              stroke="#9CA3AF"
              strokeWidth={2}
              strokeDasharray="5 5"
              dot={false}
              name="前期"
            />
          )}
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

### 高機能データテーブル

```typescript
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  getPaginationRowModel,
  flexRender,
  ColumnDef,
  SortingState,
  ColumnFiltersState
} from '@tanstack/react-table';
import { useVirtualizer } from '@tanstack/react-virtual';

interface DataTableProps<T> {
  data: T[];
  columns: ColumnDef<T>[];
  enableVirtualization?: boolean;
}

export function DataTable<T>({
  data,
  columns,
  enableVirtualization = false
}: DataTableProps<T>) {
  const [sorting, setSorting] = useState<SortingState>([]);
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([]);
  const [globalFilter, setGlobalFilter] = useState('');

  const table = useReactTable({
    data,
    columns,
    state: { sorting, columnFilters, globalFilter },
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel()
  });

  const { rows } = table.getRowModel();

  // 仮想スクロール設定
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 10
  });

  return (
    <div className="bg-white rounded-lg shadow">
      {/* フィルター */}
      <div className="p-4 border-b">
        <input
          type="text"
          placeholder="検索..."
          value={globalFilter}
          onChange={(e) => setGlobalFilter(e.target.value)}
          className="border rounded px-3 py-2 w-64"
        />
      </div>

      {/* テーブル */}
      <div ref={parentRef} className="overflow-auto max-h-[600px]">
        <table className="w-full">
          <thead className="sticky top-0 bg-gray-50">
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map(header => (
                  <th
                    key={header.id}
                    className="px-4 py-3 text-left cursor-pointer"
                    onClick={header.column.getToggleSortingHandler()}
                  >
                    {flexRender(header.column.columnDef.header, header.getContext())}
                    {{ asc: ' ↑', desc: ' ↓' }[header.column.getIsSorted() as string] ?? ''}
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody>
            {enableVirtualization ? (
              <>
                <tr style={{ height: virtualizer.getVirtualItems()[0]?.start ?? 0 }} />
                {virtualizer.getVirtualItems().map(virtualRow => {
                  const row = rows[virtualRow.index];
                  return (
                    <tr key={row.id} className="border-b hover:bg-gray-50">
                      {row.getVisibleCells().map(cell => (
                        <td key={cell.id} className="px-4 py-3">
                          {flexRender(cell.column.columnDef.cell, cell.getContext())}
                        </td>
                      ))}
                    </tr>
                  );
                })}
              </>
            ) : (
              rows.map(row => (
                <tr key={row.id} className="border-b hover:bg-gray-50">
                  {row.getVisibleCells().map(cell => (
                    <td key={cell.id} className="px-4 py-3">
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              ))
            )}
          </tbody>
        </table>
      </div>

      {/* ページネーション */}
      <TablePagination table={table} />
    </div>
  );
}
```

### PDFエクスポート

```typescript
import jsPDF from 'jspdf';
import html2canvas from 'html2canvas';

async function exportDashboardToPDF(dashboardRef: HTMLElement) {
  const canvas = await html2canvas(dashboardRef, {
    scale: 2,
    useCORS: true,
    logging: false
  });

  const imgData = canvas.toDataURL('image/png');
  const pdf = new jsPDF({
    orientation: 'landscape',
    unit: 'mm',
    format: 'a4'
  });

  const pageWidth = pdf.internal.pageSize.getWidth();
  const pageHeight = pdf.internal.pageSize.getHeight();
  const imgWidth = canvas.width;
  const imgHeight = canvas.height;

  const ratio = Math.min(pageWidth / imgWidth, pageHeight / imgHeight);
  const imgX = (pageWidth - imgWidth * ratio) / 2;
  const imgY = 10;

  pdf.addImage(imgData, 'PNG', imgX, imgY, imgWidth * ratio, imgHeight * ratio);
  pdf.save(`dashboard-${new Date().toISOString().slice(0, 10)}.pdf`);
}
```

---

## データ集計

```typescript
// サーバー側：時系列データ集計
async function getTimeSeriesData(
  metric: string,
  startDate: Date,
  endDate: Date,
  granularity: 'hour' | 'day' | 'week' | 'month'
) {
  const dateFormat = {
    hour: "YYYY-MM-DD HH24:00",
    day: "YYYY-MM-DD",
    week: "IYYY-IW",
    month: "YYYY-MM"
  }[granularity];

  const result = await prisma.$queryRaw`
    SELECT
      to_char(created_at, ${dateFormat}) as period,
      SUM(${metric}) as value,
      COUNT(*) as count
    FROM analytics_events
    WHERE created_at BETWEEN ${startDate} AND ${endDate}
    GROUP BY period
    ORDER BY period
  `;

  return result;
}
```

---

## 受け入れ基準

- [ ] 複数種類のグラフが表示される
- [ ] リアルタイムでデータが更新される
- [ ] 大量データでもスムーズに動作する
- [ ] フィルター・ソートが機能する
- [ ] CSV/PDFエクスポートができる
- [ ] レスポンシブデザイン

---

**最終更新**: 2025-10-22
