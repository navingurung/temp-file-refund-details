```javascript

"use client";

import * as React from "react";
import type { DateRange } from "react-day-picker";
import {
  CartesianGrid,
  Line,
  LineChart,
  XAxis,
  YAxis,
} from "recharts";

import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  type ChartConfig,
} from "@/components/ui/chart";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuGroup,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";

/* ----------------------------- */
/* Types */
/* ----------------------------- */

export type TaxRecord = {
  date: string;
  totalTax: number;
  totalAmount: number;
  count?: number;
};

type MetricKey = "totalAmount" | "totalTax" | "count";
type GroupBy = "day" | "week" | "month";

type ChartAreaInteractiveProps = {
  data: TaxRecord[];
  date?: DateRange;
};

type ChartPoint = {
  label: string;
  current: number;
  previous?: number;
};

/* ----------------------------- */
/* Labels */
/* ----------------------------- */

const metricLabels: Record<MetricKey, string> = {
  totalAmount: "免税売上高（税別）",
  totalTax: "還付消費税額",
  count: "免税件数",
};

const chartConfig = {
  current: { label: "現在", color: "var(--chart-1)" },
  previous: { label: "前期間", color: "var(--chart-2)" },
} satisfies ChartConfig;

/* ----------------------------- */
/* Helper Functions */
/* ----------------------------- */

function startOfDay(date: Date) {
  return new Date(date.getFullYear(), date.getMonth(), date.getDate());
}

function endOfDay(date: Date) {
  return new Date(
    date.getFullYear(),
    date.getMonth(),
    date.getDate(),
    23,
    59,
    59,
    999,
  );
}

function addDays(date: Date, days: number) {
  const next = new Date(date);
  next.setDate(next.getDate() + days);
  return next;
}

function addMonths(date: Date, months: number) {
  return new Date(date.getFullYear(), date.getMonth() + months, 1);
}

function startOfWeek(date: Date) {
  const d = startOfDay(date);
  const day = d.getDay(); // 0=Sun
  const diff = day === 0 ? -6 : 1 - day; // Monday start
  return addDays(d, diff);
}

function startOfMonth(date: Date) {
  return new Date(date.getFullYear(), date.getMonth(), 1);
}

function daysBetween(from: Date, to: Date) {
  const ms = endOfDay(to).getTime() - startOfDay(from).getTime();
  return Math.floor(ms / (1000 * 60 * 60 * 24)) + 1;
}

function toDateKey(date: Date) {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, "0");
  const d = String(date.getDate()).padStart(2, "0");
  return `${y}-${m}-${d}`;
}

function toWeekKey(date: Date) {
  return toDateKey(startOfWeek(date));
}

function toMonthKey(date: Date) {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, "0");
  return `${y}-${m}`;
}

function getMetricValue(record: TaxRecord, metric: MetricKey) {
  if (metric === "count") return record.count ?? 0;
  return record[metric] ?? 0;
}

function formatMetricValue(value: number, metric: MetricKey) {
  if (metric === "count") {
    return `${Math.round(value).toLocaleString("ja-JP")}件`;
  }
  return `¥${Math.round(value).toLocaleString("ja-JP")}`;
}

function formatYAxisTick(value: number, metric: MetricKey) {
  if (metric === "count") {
    return `${Number(value).toLocaleString("ja-JP")}件`;
  }
  if (Number(value) >= 1_000_000) {
    return `¥${(Number(value) / 1_000_000).toFixed(1)}M`;
  }
  if (Number(value) >= 1_000) {
    return `¥${(Number(value) / 1_000).toFixed(0)}K`;
  }
  return `¥${Number(value).toLocaleString("ja-JP")}`;
}

function sumValues(values: number[]) {
  return values.reduce((acc, value) => acc + value, 0);
}

function getChangePercent(current: number, previous: number) {
  if (previous === 0) return current === 0 ? 0 : 100;
  return ((current - previous) / previous) * 100;
}

function filterByRange(data: TaxRecord[], from: Date, to: Date) {
  const start = startOfDay(from).getTime();
  const end = endOfDay(to).getTime();

  return data.filter((item) => {
    const time = new Date(item.date).getTime();
    return time >= start && time <= end;
  });
}

function getPreviousRange(from: Date, to: Date) {
  const totalDays = daysBetween(from, to);
  const previousTo = addDays(startOfDay(from), -1);
  const previousFrom = addDays(previousTo, -(totalDays - 1));

  return { from: previousFrom, to: previousTo };
}

function buildSeriesMap(
  data: TaxRecord[],
  metric: MetricKey,
  groupBy: GroupBy,
) {
  const map = new Map<string, number>();

  data.forEach((record) => {
    const recordDate = new Date(record.date);

    const key =
      groupBy === "day"
        ? toDateKey(recordDate)
        : groupBy === "week"
          ? toWeekKey(recordDate)
          : toMonthKey(recordDate);

    map.set(key, (map.get(key) ?? 0) + getMetricValue(record, metric));
  });

  return map;
}

function buildDayPoints(
  currentData: TaxRecord[],
  previousData: TaxRecord[],
  metric: MetricKey,
  from: Date,
  to: Date,
): ChartPoint[] {
  const currentMap = buildSeriesMap(currentData, metric, "day");
  const previousMap = buildSeriesMap(previousData, metric, "day");
  const totalDays = daysBetween(from, to);
  const previousRange = getPreviousRange(from, to);

  return Array.from({ length: totalDays }, (_, index) => {
    const currentDate = addDays(startOfDay(from), index);
    const previousDate = addDays(startOfDay(previousRange.from), index);

    return {
      label: `${currentDate.getMonth() + 1}/${currentDate.getDate()}`,
      current: currentMap.get(toDateKey(currentDate)) ?? 0,
      previous: previousMap.get(toDateKey(previousDate)) ?? 0,
    };
  });
}

function buildWeekPoints(
  currentData: TaxRecord[],
  previousData: TaxRecord[],
  metric: MetricKey,
  from: Date,
  to: Date,
): ChartPoint[] {
  const currentMap = buildSeriesMap(currentData, metric, "week");
  const previousMap = buildSeriesMap(previousData, metric, "week");
  const previousRange = getPreviousRange(from, to);

  const currentStart = startOfWeek(from);
  const currentEnd = startOfWeek(to);
  const previousStart = startOfWeek(previousRange.from);

  const points: ChartPoint[] = [];
  let cursor = currentStart;
  let previousCursor = previousStart;

  while (cursor.getTime() <= currentEnd.getTime()) {
    points.push({
      label: `${cursor.getMonth() + 1}/${cursor.getDate()}`,
      current: currentMap.get(toWeekKey(cursor)) ?? 0,
      previous: previousMap.get(toWeekKey(previousCursor)) ?? 0,
    });

    cursor = addDays(cursor, 7);
    previousCursor = addDays(previousCursor, 7);
  }

  return points;
}

function buildMonthPoints(
  currentData: TaxRecord[],
  previousData: TaxRecord[],
  metric: MetricKey,
  from: Date,
  to: Date,
): ChartPoint[] {
  const currentMap = buildSeriesMap(currentData, metric, "month");
  const previousMap = buildSeriesMap(previousData, metric, "month");
  const previousRange = getPreviousRange(from, to);

  const currentStart = startOfMonth(from);
  const currentEnd = startOfMonth(to);
  const previousStart = startOfMonth(previousRange.from);

  const points: ChartPoint[] = [];
  let cursor = currentStart;
  let previousCursor = previousStart;

  while (cursor.getTime() <= currentEnd.getTime()) {
    points.push({
      label: `${cursor.getMonth() + 1}月`,
      current: currentMap.get(toMonthKey(cursor)) ?? 0,
      previous: previousMap.get(toMonthKey(previousCursor)) ?? 0,
    });

    cursor = addMonths(cursor, 1);
    previousCursor = addMonths(previousCursor, 1);
  }

  return points;
}

/* ----------------------------- */
/* Component */
/* ----------------------------- */

export function ChartAreaInteractive({
  data,
  date,
}: ChartAreaInteractiveProps) {
  const [metric, setMetric] = React.useState<MetricKey>("totalAmount");
  const [groupBy, setGroupBy] = React.useState<GroupBy>("day");
  const [showComparison, setShowComparison] = React.useState(false);

  // W / M / Y display labels
  const periodLabels = React.useMemo(() => {
    if (groupBy === "day") {
      return { current: "今週", previous: "前週" };
    }
    if (groupBy === "week") {
      return { current: "今月", previous: "前月" };
    }
    return { current: "今年", previous: "前年" };
  }, [groupBy]);

  // Follow DatePicker range exactly
  const effectiveRange = React.useMemo(() => {
    if (date?.from && date?.to) {
      return {
        from: startOfDay(date.from),
        to: endOfDay(date.to),
      };
    }

    if (data.length === 0) {
      const today = new Date();
      return {
        from: startOfDay(today),
        to: endOfDay(today),
      };
    }

    const times = data.map((item) => new Date(item.date).getTime());

    return {
      from: startOfDay(new Date(Math.min(...times))),
      to: endOfDay(new Date(Math.max(...times))),
    };
  }, [data, date]);

  const currentData = React.useMemo(() => {
    return filterByRange(data, effectiveRange.from, effectiveRange.to);
  }, [data, effectiveRange]);

  const previousRange = React.useMemo(() => {
    return getPreviousRange(effectiveRange.from, effectiveRange.to);
  }, [effectiveRange]);

  const previousData = React.useMemo(() => {
    return filterByRange(data, previousRange.from, previousRange.to);
  }, [data, previousRange]);

  const comparisonData = React.useMemo(() => {
    if (groupBy === "day") {
      return buildDayPoints(
        currentData,
        previousData,
        metric,
        effectiveRange.from,
        effectiveRange.to,
      );
    }

    if (groupBy === "week") {
      return buildWeekPoints(
        currentData,
        previousData,
        metric,
        effectiveRange.from,
        effectiveRange.to,
      );
    }

    return buildMonthPoints(
      currentData,
      previousData,
      metric,
      effectiveRange.from,
      effectiveRange.to,
    );
  }, [currentData, previousData, metric, groupBy, effectiveRange]);

  const currentTotal = React.useMemo(() => {
    return sumValues(currentData.map((item) => getMetricValue(item, metric)));
  }, [currentData, metric]);

  const previousTotal = React.useMemo(() => {
    return sumValues(previousData.map((item) => getMetricValue(item, metric)));
  }, [previousData, metric]);

  const changePercent = React.useMemo(() => {
    return getChangePercent(currentTotal, previousTotal);
  }, [currentTotal, previousTotal]);

  return (
    <Card>
      <CardHeader className="flex flex-row items-start justify-between gap-3 pb-2">
        <div className="space-y-2">
          <CardTitle>推移</CardTitle>

          <DropdownMenu>
            <DropdownMenuTrigger asChild>
              <Button variant="outline" className="h-9 min-w-[180px] justify-center">
                {metricLabels[metric]}
              </Button>
            </DropdownMenuTrigger>

            <DropdownMenuContent align="start">
              <DropdownMenuGroup>
                <DropdownMenuItem onClick={() => setMetric("totalAmount")}>
                  免税売上高（税別）
                </DropdownMenuItem>
                <DropdownMenuItem onClick={() => setMetric("totalTax")}>
                  還付消費税額
                </DropdownMenuItem>
                <DropdownMenuItem onClick={() => setMetric("count")}>
                  免税件数
                </DropdownMenuItem>
              </DropdownMenuGroup>
            </DropdownMenuContent>
          </DropdownMenu>
        </div>

        <div className="flex items-center gap-3">
          <div className="inline-flex rounded-md border p-0.5">
            <Button
              size="sm"
              variant={groupBy === "day" ? "secondary" : "ghost"}
              className="h-7 px-2 text-xs"
              onClick={() => setGroupBy("day")}
            >
              W
            </Button>
            <Button
              size="sm"
              variant={groupBy === "week" ? "secondary" : "ghost"}
              className="h-7 px-2 text-xs"
              onClick={() => setGroupBy("week")}
            >
              M
            </Button>
            <Button
              size="sm"
              variant={groupBy === "month" ? "secondary" : "ghost"}
              className="h-7 px-2 text-xs"
              onClick={() => setGroupBy("month")}
            >
              Y
            </Button>
          </div>

          <label className="flex items-center gap-1.5 whitespace-nowrap text-xs text-muted-foreground">
            <Checkbox
              id="comparison"
              checked={showComparison}
              onCheckedChange={(checked) => setShowComparison(!!checked)}
            />
            前期間と比較
          </label>
        </div>
      </CardHeader>

      <CardContent className="space-y-3">
        {showComparison && (
          <div className="grid grid-cols-1 gap-2 sm:grid-cols-3">
            <div className="rounded-md border px-3 py-2">
              <p className="text-xs text-muted-foreground">{periodLabels.current}</p>
              <p className="text-lg font-semibold">
                {formatMetricValue(currentTotal, metric)}
              </p>
            </div>
            <div className="rounded-md border px-3 py-2">
              <p className="text-xs text-muted-foreground">{periodLabels.previous}</p>
              <p className="text-lg font-semibold">
                {formatMetricValue(previousTotal, metric)}
              </p>
            </div>
            <div className="rounded-md border px-3 py-2">
              <p className="text-xs text-muted-foreground">増減率</p>
              <p
                className={`text-lg font-semibold ${
                  changePercent >= 0 ? "text-emerald-600" : "text-rose-600"
                }`}
              >
                {changePercent > 0 ? "+" : ""}
                {changePercent.toFixed(1)}%
              </p>
            </div>
          </div>
        )}

        <ChartContainer config={chartConfig} className="h-[260px] w-full">
          <LineChart data={comparisonData}>
            <CartesianGrid vertical={false} />

            <XAxis
              dataKey="label"
              tickLine={false}
              axisLine={false}
              tickMargin={8}
              minTickGap={20}
            />

            <YAxis
              tickLine={false}
              axisLine={false}
              width={70}
              tickFormatter={(value) => formatYAxisTick(Number(value), metric)}
            />

            <ChartTooltip
              cursor={false}
              content={
                <ChartTooltipContent
                  indicator="line"
                  formatter={(value, name) => {
                    return [formatMetricValue(Number(value), metric), String(name)];
                  }}
                />
              }
            />

            <Line
              dataKey="current"
              name={periodLabels.current}
              type="natural"
              stroke="var(--color-current)"
              strokeWidth={2.5}
              strokeLinecap="round"
              strokeLinejoin="round"
              dot={false}
              activeDot={{ r: 4 }}
              isAnimationActive
              animationDuration={500}
              animationEasing="ease"
            />

            {showComparison && (
              <Line
                dataKey="previous"
                name={periodLabels.previous}
                type="natural"
                stroke="var(--color-previous)"
                strokeWidth={2}
                strokeDasharray="6 6"
                strokeLinecap="round"
                strokeLinejoin="round"
                dot={false}
                activeDot={{ r: 4 }}
                isAnimationActive
                animationDuration={500}
                animationEasing="ease"
              />
            )}
          </LineChart>
        </ChartContainer>
      </CardContent>
    </Card>
  );
}

```
