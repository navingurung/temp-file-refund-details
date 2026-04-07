# temp-file-refund-details

"use client";

import { useState } from "react";
import axios from "axios";
import Link from "next/link";
import { toast } from "sonner";

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
import React from "react";

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

// 2026-02-20T06:56:19.663114(UTC)を2026/02/20 15:56:19(JST)の形式に変換する関数
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

// YYYYMMDDをYYYY/MM/DDに変換する関数
const formatDateCompact = (dateStr: string) => {
  if (dateStr.length !== 8) return dateStr;
  return `${dateStr.slice(0, 4)}/${dateStr.slice(4, 6)}/${dateStr.slice(6)}`;
};

export default function RefundDetailClient({ refund }: { refund: Refund }) {
  const [refundState, setRefundState] = useState<Refund>(refund);
  const [isDeleting, setIsDeleting] = useState(false);
  const [open, setOpen] = useState(false);

  if (!refundState) {
    return null;
  }

  const handleDelete = async () => {
    if (isDeleting) return;

    try {
      setIsDeleting(true);

      // test API call only
      await axios.delete("https://fakestoreapi.com/products/1");

      setRefundState((prev) => ({
        ...prev,
        status: "cancelled",
        deleted_at: new Date().toISOString(),
        delete_request_body: JSON.stringify(
          {
            id: prev.id,
            action: "delete",
          },
          null,
          2,
        ),
        delete_response_body: JSON.stringify(
          {
            success: true,
            message: "Test delete completed",
          },
          null,
          2,
        ),
      }));

      toast.success("申請を取り消しました");
      setOpen(false);
    } catch (error) {
      console.error(error);
      toast.error("取消処理に失敗しました");
    } finally {
      setIsDeleting(false);
    }
  };

  return (
    <div className="max-w-3xl w-full mx-auto p-6 space-y-6">
      {/* ヘッダー */}
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">免税申請詳細</h1>
      </div>

      <div>
        <ID label="申請ID" value={refundState.id} />
        <ID label="作成日時" value={formatDate(refundState.created_at)} />
        {refundState.deleted_at && (
          <ID label="取消日時" value={formatDate(refundState.deleted_at)} />
        )}
        <Badge variant={STATUS[refundState.status]?.variant as any}>
          {STATUS[refundState.status]?.label || refundState.status}
        </Badge>
      </div>

      {/* パスポート情報 */}
      <Card>
        <CardContent className="space-y-4 pt-6">
          <h2 className="font-semibold">パスポート情報</h2>
          <Separator />
          <Info label="パスポート番号" value={refundState.passport_no} />
          <Info label="氏名" value={refundState.name} />
          <Info label="国籍" value={refundState.nation} />
          <Info
            label="生年月日"
            value={formatDateCompact(refundState.birth_date)}
          />
          <Info
            label="在留資格"
            value={landStatusMap[refundState.land_status]}
          />
          <Info label="入国日" value={formatDateCompact(refundState.land_date)} />
          {refundState.residence_country && (
            <Info label="居住国" value={refundState.residence_country} />
          )}
          {refundState.chinese_state && (
            <Info label="居住州" value={refundState.chinese_state} />
          )}
        </CardContent>
      </Card>

      {/* 金額情報 */}
      <Card>
        <CardContent className="space-y-4 pt-6">
          <h2 className="font-semibold">注文情報</h2>
          <Separator />
          <Info label="注文ID" value={refundState.order_id || "-"} />
          <Info
            label="合計（税込）"
            value={`¥${(refundState.total_received ?? 0).toLocaleString()}`}
          />
          <Info
            label="消費税額"
            value={`¥${(refundState.total_tax ?? 0).toLocaleString()}`}
          />
        </CardContent>
      </Card>

      {/* NTA送信情報 */}
      <Card>
        <CardContent className="space-y-4 pt-6">
          <h2 className="font-semibold">NTA送信情報</h2>
          <Separator />
          <AccordionDemo
            sent_request_json={refundState.request_body}
            response_request_json={refundState.response_body}
            sent_delete_json={refundState.delete_request_body}
            response_delete_json={refundState.delete_response_body}
          />
        </CardContent>
      </Card>

      {/* アクション */}
      <div className="flex justify-between pt-4">
        <Button variant="outline" asChild>
          <Link href="/dashboard">ダッシュボードへ戻る</Link>
        </Button>

        <Dialog open={open} onOpenChange={setOpen}>
          <DialogTrigger asChild>
            <Button
              variant="destructive"
              disabled={isDeleting || !!refundState.deleted_at}
            >
              {refundState.deleted_at ? "取消済み" : "申請を取り消す"}
            </Button>
          </DialogTrigger>

          <DialogContent className="sm:max-w-md">
            <DialogClose asChild>
              <button
                className="absolute right-4 top-4 rounded-sm opacity-70 transition-opacity hover:opacity-100"
                aria-label="閉じる"
              >
                <X className="h-4 w-4" />
              </button>
            </DialogClose>

            <DialogHeader>
              <DialogTitle>この申請を取り消しますか？</DialogTitle>
              <DialogDescription>
                取り消し後、この申請は取消済みとして記録されます。
              </DialogDescription>
            </DialogHeader>

            <DialogFooter className="flex-col-reverse gap-2 sm:flex-row sm:justify-end">
              <DialogClose asChild>
                <Button variant="outline">キャンセル</Button>
              </DialogClose>

              <Button
                variant="destructive"
                onClick={handleDelete}
                disabled={isDeleting}
              >
                {isDeleting ? "取消中..." : "取り消す"}
              </Button>
            </DialogFooter>
          </DialogContent>
        </Dialog>
      </div>
    </div>
  );
}

/* ---------- 小コンポーネント ---------- */

function Info({ label, value }: { label: string; value: string }) {
  return (
    <div className="flex justify-between text-sm">
      <span className="text-muted-foreground">{label}</span>
      <span className="font-medium">{value}</span>
    </div>
  );
}

function ID({ label, value }: { label: string; value: string }) {
  return (
    <div className="flex gap-2 text-sm">
      <span className="text-muted-foreground">{label}:</span>
      <span className="font-medium">{value}</span>
    </div>
  );
}
