```javascript
"use client";

import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";
import { Separator } from "@/components/ui/separator";
import { STATUS, type Status } from "@/lib/constants/status";
import { AccordionDemo } from "@/app/dashboard/refund/[id]/accordion";
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
  DialogClose,
} from "@/components/ui/dialog";
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from "@/components/ui/tooltip";
import {
  ShieldCheck,
  ClipboardList,
  RefreshCw,
  CheckCircle2,
  XCircle,
  Ban,
} from "lucide-react";
import Link from "next/link";
import { useState } from "react";
import { cn } from "@/lib/utils";

type Refund = {
  id: string;
  status: Status;
  created_at: string;
  passport_no: string;
  name: string;
  nation: string;
  birth_date: string;
  land_status: string;
  land_date: string;
  email: string;
  residence_country: string;
  chinese_state: string;
  total_tax: number;
  total_received: number;
  order_id: string;
  request_body?: string;
  response_body?: string;
  delete_request_body?: string;
  delete_response_body?: string;
  deleted_at?: string;
};

const formatDate = (dateStr: string) => {
  const date = new Date(dateStr);
  return date.toLocaleString("ja-JP", {
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  });
};

const landStatusMap: Record<string, string> = {
  "11": "短期滞在",
  "14": "外交",
  "17": "公用",
  "99": "その他",
  "96": "米軍構成員",
  "91": "上陸許可書による入国",
  "95": "非居住者に該当する日本国籍の者",
};

const formatDateCompact = (dateStr: string) => {
  if (dateStr.length !== 8) return dateStr;
  return `${dateStr.slice(0, 4)}/${dateStr.slice(4, 6)}/${dateStr.slice(6)}`;
};

type TrackerVariant = "success" | "warning" | "info" | "cancelled";
type TrackerOutcome = "rejected" | "cancelled" | null;

type TrackerStep = {
  label: string;
  timeline?: string;
  icon: React.ComponentType<{ className?: string }>;
  variant: TrackerVariant;
  status: "completed" | "current" | "pending" | "terminal";
  animateLine?: boolean;
};

type TrackerModel = {
  steps: TrackerStep[];
  outcome: TrackerOutcome;
};

function getVariantStyles(variant: TrackerVariant) {
  switch (variant) {
    case "success":
      return {
        currentCircle: "border-emerald-500 bg-emerald-500 text-white",
        currentText: "text-emerald-700",
        line: "bg-emerald-500",
        lineSoft: "bg-emerald-100",
        flowText: "text-emerald-500",
      };

    case "warning":
      return {
        currentCircle: "border-amber-500 bg-amber-50 text-amber-600",
        currentText: "text-amber-700",
        line: "bg-amber-500",
        lineSoft: "bg-amber-100",
        flowText: "text-amber-500",
      };

    case "cancelled":
      return {
        currentCircle: "border-rose-500 bg-rose-50 text-rose-600",
        currentText: "text-rose-700",
        line: "bg-rose-500",
        lineSoft: "bg-rose-100",
        flowText: "text-rose-500",
      };

    case "info":
    default:
      return {
        currentCircle: "border-sky-500 bg-sky-50 text-sky-600",
        currentText: "text-sky-700",
        line: "bg-sky-500",
        lineSoft: "bg-sky-100",
        flowText: "text-sky-500",
      };
  }
}

function buildTrackerModel(status: Status): TrackerModel {
  const label = STATUS[status]?.label;

  const baseSteps: TrackerStep[] = [
    {
      label: "税関審査待ち",
      timeline: "通常 1〜2週間",
      icon: ShieldCheck,
      variant: STATUS.customs_review_pending.variant as TrackerVariant,
      status: "pending",
    },
    {
      label: "処理待ち",
      timeline: "通常 数日程度",
      icon: ClipboardList,
      variant: STATUS.trj_customer_registration_pending.variant as TrackerVariant,
      status: "pending",
    },
    {
      label: "処理中",
      timeline: "通常 3〜5営業日",
      icon: RefreshCw,
      variant: STATUS.trj_transaction_registration_pending.variant as TrackerVariant,
      status: "pending",
    },
    {
      label: "送金済",
      icon: CheckCircle2,
      variant: STATUS.completed.variant as TrackerVariant,
      status: "pending",
    },
  ];

  if (status === "customs_rejected" || label === "免税不可") {
    return {
      steps: [
        {
          label: "免税不可",
          icon: XCircle,
          variant: "cancelled",
          status: "terminal",
        },
        { ...baseSteps[1] },
        { ...baseSteps[2] },
        { ...baseSteps[3] },
      ],
      outcome: "rejected",
    };
  }

  if (status === "cancelled" || status === "refunded" || label === "キャンセル") {
    return {
      steps: [
        { ...baseSteps[0], status: "completed" },
        { ...baseSteps[1], status: "completed" },
        {
          label: "キャンセル",
          icon: Ban,
          variant: "cancelled",
          status: "terminal",
        },
        { ...baseSteps[3] },
      ],
      outcome: "cancelled",
    };
  }

  switch (label) {
    case "税関審査待ち":
      return {
        steps: [
          { ...baseSteps[0], status: "current", animateLine: true },
          { ...baseSteps[1] },
          { ...baseSteps[2] },
          { ...baseSteps[3] },
        ],
        outcome: null,
      };

    case "処理待ち":
      return {
        steps: [
          { ...baseSteps[0], status: "completed" },
          { ...baseSteps[1], status: "current", animateLine: true },
          { ...baseSteps[2] },
          { ...baseSteps[3] },
        ],
        outcome: null,
      };

    case "処理中":
    case "処理中(WISE)":
    case "資金戻り(WISE)":
      return {
        steps: [
          { ...baseSteps[0], status: "completed" },
          { ...baseSteps[1], status: "completed" },
          { ...baseSteps[2], status: "current", animateLine: true },
          { ...baseSteps[3] },
        ],
        outcome: null,
      };

    case "送金済":
      return {
        steps: [
          { ...baseSteps[0], status: "completed" },
          { ...baseSteps[1], status: "completed" },
          { ...baseSteps[2], status: "completed" },
          { ...baseSteps[3], status: "current" },
        ],
        outcome: null,
      };

    default:
      return {
        steps: [
          { ...baseSteps[0], status: "current", animateLine: true },
          { ...baseSteps[1] },
          { ...baseSteps[2] },
          { ...baseSteps[3] },
        ],
        outcome: null,
      };
  }
}

export default function RefundDetailClient({ refund }: { refund: Refund }) {
  if (!refund) return null;

  const [isDeleting, setIsDeleting] = useState(false);
  const [open, setOpen] = useState(false);

  const cancelableStatuses: Status[] = [
    "requested",
    "customs_review_pending",
    "trj_customer_registration_pending",
    "trj_transaction_registration_pending",
  ];

  const canCancel = cancelableStatuses.includes(refund.status);
  const disable = !canCancel || isDeleting;

  const trackerState = buildTrackerModel(refund.status);

  return (
    <div className="w-full max-w-6xl mx-auto px-4 py-6 sm:px-6 lg:px-8 space-y-6">
      <div className="space-y-3">
        <h1 className="text-2xl font-bold">免税申請詳細</h1>

        <div className="flex flex-col gap-2 sm:flex-row sm:items-center sm:justify-between">
          <div className="flex flex-col gap-1 text-sm sm:flex-row sm:items-center sm:gap-8">
            <ID label="申請ID" value={refund.id} />
            <ID label="作成日時" value={formatDate(refund.created_at)} />
            {refund.deleted_at && (
              <ID label="取消日時" value={formatDate(refund.deleted_at)} />
            )}
          </div>

          <Badge variant={STATUS[refund.status]?.variant as any}>
            {STATUS[refund.status]?.label || refund.status}
          </Badge>
        </div>
      </div>

      <Card>
        <CardContent className="pt-4">
          <RefundTracker steps={trackerState.steps} />

          {trackerState.outcome === "rejected" && (
            <div className="mt-5">
              <OutcomeCard
                type="rejected"
                title="免税不可"
                description="税関審査の結果、免税対象外となりました。"
              />
            </div>
          )}

          {trackerState.outcome === "cancelled" && (
            <div className="mt-5">
              <OutcomeCard
                type="cancelled"
                title="キャンセル"
                description="この申請は途中で取り消されました。"
              />
            </div>
          )}
        </CardContent>
      </Card>

      <div className="grid grid-cols-1 gap-6 lg:grid-cols-2">
        <Card>
          <CardContent className="space-y-4 pt-6">
            <h2 className="font-semibold">パスポート情報</h2>
            <Separator />
            <Info label="パスポート番号" value={refund.passport_no} />
            <Info label="氏名" value={refund.name} />
            <Info label="国籍" value={refund.nation} />
            <Info label="生年月日" value={formatDateCompact(refund.birth_date)} />
            <Info label="在留資格" value={landStatusMap[refund.land_status] || "-"} />
            <Info label="入国日" value={formatDateCompact(refund.land_date)} />
            {refund.residence_country && (
              <Info label="居住国" value={refund.residence_country} />
            )}
            {refund.chinese_state && (
              <Info label="居住州" value={refund.chinese_state} />
            )}
          </CardContent>
        </Card>

        <Card>
          <CardContent className="space-y-4 pt-6">
            <h2 className="font-semibold">注文情報</h2>
            <Separator />
            <Info label="注文ID" value={refund.order_id || "-"} />
            <Info
              label="合計（税込）"
              value={`¥${(refund.total_received ?? 0).toLocaleString()}`}
            />
            <Info
              label="消費税額"
              value={`¥${(refund.total_tax ?? 0).toLocaleString()}`}
            />
          </CardContent>
        </Card>
      </div>

      <Card>
        <CardContent className="space-y-4 pt-6">
          <h2 className="font-semibold">NTA送信情報</h2>
          <Separator />
          <AccordionDemo
            sent_request_json={refund.request_body}
            response_request_json={refund.response_body}
            sent_delete_json={refund.delete_request_body}
            response_delete_json={refund.delete_response_body}
          />
        </CardContent>
      </Card>

      <div className="flex flex-col gap-3 pt-2 sm:flex-row sm:justify-between">
        <Button variant="outline" asChild>
          <Link href="/dashboard">ダッシュボードへ戻る</Link>
        </Button>

        <TooltipProvider>
          <Tooltip>
            <TooltipTrigger asChild>
              <span tabIndex={disable ? 0 : -1}>
                <Dialog open={open} onOpenChange={setOpen}>
                  <DialogTrigger asChild>
                    <Button variant="destructive" disabled={disable}>
                      申請を取り消す
                    </Button>
                  </DialogTrigger>

                  <DialogContent className="sm:max-w-md">
                    <DialogHeader>
                      <DialogTitle>この申請を取り消しますか？</DialogTitle>
                      <DialogDescription>
                        取り消し後、この申請はキャンセルとして記録されます。
                      </DialogDescription>
                    </DialogHeader>

                    <DialogFooter className="flex-col-reverse gap-2 sm:flex-row sm:justify-end">
                      <DialogClose asChild>
                        <Button variant="outline">キャンセル</Button>
                      </DialogClose>

                      <Button variant="destructive" disabled={isDeleting}>
                        {isDeleting ? "取消中..." : "取り消す"}
                      </Button>
                    </DialogFooter>
                  </DialogContent>
                </Dialog>
              </span>
            </TooltipTrigger>

            {!canCancel && !isDeleting && (
              <TooltipContent>
                <p>このステータスでは、申請を取り消すことができません。</p>
              </TooltipContent>
            )}
          </Tooltip>
        </TooltipProvider>
      </div>
    </div>
  );
}

function RefundTracker({ steps }: { steps: TrackerStep[] }) {
  return (
    <div className="w-full">
      <div className="hidden md:grid md:grid-cols-4 md:gap-6">
        {steps.map((step, index) => {
          const Icon = step.icon;
          const styles = getVariantStyles(step.variant);
          const isCompleted = step.status === "completed";
          const isCurrent = step.status === "current";
          const isPending = step.status === "pending";
          const isTerminal = step.status === "terminal";
          const isLast = index === steps.length - 1;

          return (
            <div key={`${step.label}-${index}`} className="relative">
              <div className="flex flex-col items-center text-center gap-2">
                <div
                  className={cn(
                    "relative z-10 flex h-12 w-12 items-center justify-center rounded-full border-2 transition-all duration-300",
                    isCurrent && styles.currentCircle,
                    isTerminal && styles.currentCircle,
                    isCompleted &&
                      "border-slate-300 bg-slate-50 text-slate-400",
                    isPending && "border-slate-300 bg-white text-slate-400"
                  )}
                >
                  <Icon
                    className={cn(
                      "h-5 w-5",
                      step.label === "処理中" &&
                        isCurrent &&
                        "animate-spin [animation-duration:2.5s]"
                    )}
                  />
                </div>

                <p
                  className={cn(
                    "text-sm font-semibold",
                    (isCurrent || isTerminal) && styles.currentText,
                    isCompleted && "text-slate-400",
                    isPending && "text-slate-400"
                  )}
                >
                  {step.label}
                </p>

                {!isLast && !isTerminal && step.timeline && (
                  <p
                    className={cn(
                      "text-xs",
                      isCurrent ? "text-muted-foreground" : "text-slate-400"
                    )}
                  >
                    {step.timeline}
                  </p>
                )}
              </div>

              {index < steps.length - 1 && (
                <div className="absolute left-[calc(50%+28px)] top-6 h-[3px] w-[calc(100%-56px)] overflow-hidden rounded-full bg-slate-200">
                  {isCompleted && (
                    <div className="h-full w-full rounded-full bg-slate-300" />
                  )}

                  {isCurrent && step.animateLine && (
                    <div
                      className={cn(
                        "relative h-full w-full rounded-full",
                        styles.lineSoft
                      )}
                    >
                      <div
                        className={cn(
                          "tracker-flow absolute inset-y-0 w-1/2 rounded-full bg-gradient-to-r from-transparent via-current to-transparent",
                          styles.flowText
                        )}
                      />
                    </div>
                  )}

                  {(isPending || isTerminal) && (
                    <div className="h-full w-full rounded-full bg-slate-200" />
                  )}
                </div>
              )}
            </div>
          );
        })}
      </div>

      <div className="space-y-4 md:hidden">
        {steps.map((step, index) => {
          const Icon = step.icon;
          const styles = getVariantStyles(step.variant);
          const isCompleted = step.status === "completed";
          const isCurrent = step.status === "current";
          const isPending = step.status === "pending";
          const isTerminal = step.status === "terminal";
          const isLast = index === steps.length - 1;

          return (
            <div
              key={`${step.label}-${index}`}
              className={cn(
                "flex items-start gap-3 rounded-xl border p-4 transition-all duration-300",
                (isCurrent || isTerminal) && "border-slate-200 bg-slate-50/50",
                isCompleted && "border-slate-200 bg-slate-50/40",
                isPending && "border-slate-200 bg-white"
              )}
            >
              <div
                className={cn(
                  "flex h-10 w-10 shrink-0 items-center justify-center rounded-full border-2",
                  isCurrent && styles.currentCircle,
                  isTerminal && styles.currentCircle,
                  isCompleted &&
                    "border-slate-300 bg-slate-50 text-slate-400",
                  isPending && "border-slate-300 bg-white text-slate-400"
                )}
              >
                <Icon
                  className={cn(
                    "h-4 w-4",
                    step.label === "処理中" &&
                      isCurrent &&
                      "animate-spin [animation-duration:2.5s]"
                  )}
                />
              </div>

              <div className="min-w-0">
                <p
                  className={cn(
                    "text-sm font-semibold",
                    (isCurrent || isTerminal) && styles.currentText,
                    isCompleted && "text-slate-400",
                    isPending && "text-slate-500"
                  )}
                >
                  {step.label}
                </p>

                {!isLast && !isTerminal && step.timeline && (
                  <p
                    className={cn(
                      "mt-1 text-xs",
                      isCurrent ? "text-muted-foreground" : "text-slate-400"
                    )}
                  >
                    {step.timeline}
                  </p>
                )}
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
}

function OutcomeCard({
  type,
  title,
  description,
}: {
  type: "rejected" | "cancelled";
  title: string;
  description: string;
}) {
  const isRejected = type === "rejected";

  return (
    <div
      className={cn(
        "flex items-start gap-3 rounded-xl border px-4 py-4",
        isRejected
          ? "border-destructive/30 bg-destructive/5"
          : "border-muted-foreground/20 bg-muted/40"
      )}
    >
      <div
        className={cn(
          "mt-0.5",
          isRejected ? "text-destructive" : "text-muted-foreground"
        )}
      >
        {isRejected ? (
          <XCircle className="h-5 w-5" />
        ) : (
          <Ban className="h-5 w-5" />
        )}
      </div>

      <div className="space-y-1">
        <p
          className={cn(
            "text-sm font-semibold",
            isRejected ? "text-destructive" : "text-foreground"
          )}
        >
          {title}
        </p>
        <p className="text-sm text-muted-foreground">{description}</p>
      </div>
    </div>
  );
}

function Info({ label, value }: { label: string; value: string }) {
  return (
    <div className="flex items-center justify-between gap-4 text-sm">
      <span className="text-muted-foreground">{label}</span>
      <span className="font-medium text-right break-all">{value}</span>
    </div>
  );
}

function ID({ label, value }: { label: string; value: string }) {
  return (
    <div className="flex gap-2 text-sm">
      <span className="text-muted-foreground">{label}:</span>
      <span className="font-medium break-all">{value}</span>
    </div>
  );
}

```

```css
@keyframes tracker-flow {
  0% {
    left: -50%;
    opacity: 0.2;
  }
  50% {
    opacity: 1;
  }
  100% {
    left: 100%;
    opacity: 0.2;
  }
}

.tracker-flow {
  animation: tracker-flow 1.8s linear infinite;
}
```
