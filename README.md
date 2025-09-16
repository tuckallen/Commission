import React, { useMemo, useState } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Button } from "@/components/ui/button";

/**
 * Simple currency formatter
 */
const fmtUSD = (n: number) =>
  new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
    maximumFractionDigits: 0,
  }).format(Math.max(0, Number.isFinite(n) ? n : 0));

/**
 * Pure calculation helper (testable)
 */
export type ComparisonInputs = {
  avgLoanAmount: number; // dollars
  loansPerMonth: number; // count
  grossMode: "bps" | "dollar"; // how gross is entered
  grossBps: number; // if grossMode === "bps"
  grossDollar: number; // if grossMode === "dollar"
  structure: "bps" | "split"; // current comp type
  currentPayoutBps: number; // if structure === "bps"
  currentSplitPct: number; // if structure === "split"
  currentPerFileFee: number; // $ fee at current employer
  wefundFlatFee?: number; // default 795
};

export type ComparisonResult = {
  current: { perLoan: number; monthly: number; annual: number; note: string };
  wefund: { perLoan: number; monthly: number; annual: number; note: string };
  deltaAnnual: number; // wefund - current
};

export function computeComparison(i: ComparisonInputs): ComparisonResult {
  const WEFUND_FLAT_FEE = i.wefundFlatFee ?? 795;
  const lpm = Math.max(0, i.loansPerMonth || 0);
  const amount = Math.max(0, i.avgLoanAmount || 0);

  const grossPerLoan =
    i.grossMode === "bps"
      ? amount * ((Math.max(0, i.grossBps || 0)) / 10000)
      : Math.max(0, i.grossDollar || 0);

  const currentPerLoanRaw =
    i.structure === "bps"
      ? amount * ((Math.max(0, i.currentPayoutBps || 0)) / 10000)
      : grossPerLoan * ((Math.max(0, Math.min(100, i.currentSplitPct || 0))) / 100);

  const currentPerLoan = Math.max(0, currentPerLoanRaw - Math.max(0, i.currentPerFileFee || 0));
  const currentMonthly = currentPerLoan * lpm;
  const currentAnnual = currentMonthly * 12;

  const wefundPerLoan = Math.max(0, grossPerLoan - WEFUND_FLAT_FEE);
  const wefundMonthly = wefundPerLoan * lpm;
  const wefundAnnual = wefundMonthly * 12;

  return {
    current: {
      perLoan: currentPerLoan,
      monthly: currentMonthly,
      annual: currentAnnual,
      note:
        i.structure === "bps"
          ? `Your payout ${Math.max(0, i.currentPayoutBps || 0)} bps − $${Math.max(0, i.currentPerFileFee || 0)}/file`
          : `Your split ${Math.max(0, Math.min(100, i.currentSplitPct || 0))}% of shared gross − $${Math.max(0, i.currentPerFileFee || 0)}/file`,
    },
    wefund: {
      perLoan: wefundPerLoan,
      monthly: wefundMonthly,
      annual: wefundAnnual,
      note:
        `Same gross (${i.grossMode === "bps" ? `${Math.max(0, i.grossBps || 0)} bps` : fmtUSD(Math.max(0, i.grossDollar || 0))}) − $${WEFUND_FLAT_FEE}/file`,
    },
    deltaAnnual: wefundAnnual - currentAnnual,
  };
}

export default function CommissionComparisonCalculatorBasic() {
  // Universal assumptions
  const [avgLoanAmount, setAvgLoanAmount] = useState<number>(450000);
  const [loansPerMonth, setLoansPerMonth] = useState<number>(4);

  // Always assume the *same* gross commission per loan for both scenarios
  const [grossMode, setGrossMode] = useState<"bps" | "dollar">("bps");
  const [grossBps, setGrossBps] = useState<number>(200);
  const [grossDollar, setGrossDollar] = useState<number>(0);

  // Current shop structure
  const [structure, setStructure] = useState<"bps" | "split">("split");
  const [currentPayoutBps, setCurrentPayoutBps] = useState<number>(120);
  const [currentSplitPct, setCurrentSplitPct] = useState<number>(70);
  const [currentPerFileFee, setCurrentPerFileFee] = useState<number>(0);

  // WeFund fixed fee for the 100% plan (locked)
  const WEFUND_FLAT_FEE = 795;

  // Derived numbers via the pure helper
  const { current, wefund, deltaAnnual } = useMemo(
    () =>
      computeComparison({
        avgLoanAmount,
        loansPerMonth,
        grossMode,
        grossBps,
        grossDollar,
        structure,
        currentPayoutBps,
        currentSplitPct,
        currentPerFileFee,
        wefundFlatFee: WEFUND_FLAT_FEE,
      }),
    [
      avgLoanAmount,
      loansPerMonth,
      grossMode,
      grossBps,
      grossDollar,
      structure,
      currentPayoutBps,
      currentSplitPct,
      currentPerFileFee,
    ]
  );

  const resetAll = () => {
    setAvgLoanAmount(450000);
    setLoansPerMonth(4);
    setGrossMode("bps");
    setGrossBps(200);
    setGrossDollar(0);
    setStructure("split");
    setCurrentPayoutBps(120);
    setCurrentSplitPct(70);
    setCurrentPerFileFee(0);
  };

  return (
    <div className="w-full max-w-6xl mx-auto p-4 md:p-8">
      {/* Header */}
      <div className="rounded-2xl border bg-gradient-to-br from-[#338e3d]/10 to-emerald-50 p-5 md:p-6 flex items-center justify-between">
        <div>
          <h1 className="text-2xl md:text-3xl font-semibold tracking-tight">Commission Comparison</h1>
          <p className="text-sm text-muted-foreground mt-1">
            Same gross. Different outcomes. Compare your current payout vs
            {" "}
            <span className="text-[#338e3d] font-medium">WeFund 100% − $795</span>.
          </p>
        </div>
        <div className="hidden md:flex items-center gap-3">
          <Button variant="outline" className="rounded-xl" onClick={resetAll}>
            Reset
          </Button>
          <span className="inline-flex items-center text-xs px-2 py-1 rounded-full bg-[#338e3d]/10 text-[#338e3d] border border-[#338e3d]/30">
            WeFund Mortgage
          </span>
        </div>
      </div>

      {/* Inputs */}
      <div className="grid lg:grid-cols-3 gap-4 mt-6">
        {/* Block: Production Details */}
        <Card className="rounded-2xl shadow-sm">
          <CardContent className="p-5 space-y-4">
            <div className="flex items-center justify-between">
              <h3 className="font-medium">Production Details</h3>
              <span className="text-xs px-2 py-1 rounded-full bg-slate-100">Step 1</span>
            </div>
            <div className="space-y-2">
              <Label>Average Loan Amount</Label>
              <Input
                className="rounded-xl"
                type="number"
                value={avgLoanAmount}
                min={50000}
                onChange={(e) => setAvgLoanAmount(Number(e.target.value || 0))}
              />
            </div>
            <div className="space-y-2">
              <Label>Loans per Month</Label>
              <Input
                className="rounded-xl"
                type="number"
                value={loansPerMonth}
                min={0}
                onChange={(e) => setLoansPerMonth(Number(e.target.value || 0))}
              />
            </div>
          </CardContent>
        </Card>

        {/* Block: Your Company’s Compensation */}
        <Card className="rounded-2xl shadow-sm">
          <CardContent className="p-5 space-y-4">
            <div className="flex items-center justify-between">
              <h3 className="font-medium">Your Company’s Compensation</h3>
              <span className="text-xs px-2 py-1 rounded-full bg-slate-100">Step 2</span>
            </div>
            <div className="space-y-2">
              <Label>Average Gross Commission</Label>
              <Select value={grossMode} onValueChange={(v) => setGrossMode(v as any)}>
                <SelectTrigger className="rounded-xl">
                  <SelectValue placeholder="Select input type" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="bps">Enter gross in bps</SelectItem>
                  <SelectItem value="dollar">Enter gross as $ per loan</SelectItem>
                </SelectContent>
              </Select>
            </div>
            {grossMode === "bps" ? (
              <div className="space-y-2">
                <Label>Gross (bps)</Label>
                <Input
                  className="rounded-xl"
                  type="number"
                  value={grossBps}
                  min={0}
                  step={5}
                  onChange={(e) => setGrossBps(Number(e.target.value || 0))}
                />
              </div>
            ) : (
              <div className="space-y-2">
                <Label>Gross ($ per loan)</Label>
                <Input
                  className="rounded-xl"
                  type="number"
                  value={grossDollar}
                  min={0}
                  step={100}
                  onChange={(e) => setGrossDollar(Number(e.target.value || 0))}
                />
              </div>
            )}
            <p className="text-xs text-muted-foreground">
              We use this same gross for both your current shop and WeFund.
            </p>
          </CardContent>
        </Card>

        {/* Block: Your Commission Structure */}
        <Card className="rounded-2xl shadow-sm">
          <CardContent className="p-5 space-y-4">
            <div className="flex items-center justify-between">
              <h3 className="font-medium">Your Commission Structure</h3>
              <span className="text-xs px-2 py-1 rounded-full bg-slate-100">Step 3</span>
            </div>
            <div className="space-y-2">
              <Label>Commission Structure</Label>
              <Select value={structure} onValueChange={(v) => setStructure(v as any)}>
                <SelectTrigger className="rounded-xl">
                  <SelectValue placeholder="Choose structure" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="bps">Flat BPS Payout</SelectItem>
                  <SelectItem value="split">Split of Gross</SelectItem>
                </SelectContent>
              </Select>
            </div>

            {structure === "bps" ? (
              <div className="space-y-2">
                <Label>Your Payout (bps to you)</Label>
                <Input
                  className="rounded-xl"
                  type="number"
                  value={currentPayoutBps}
                  min={0}
                  step={5}
                  onChange={(e) => setCurrentPayoutBps(Number(e.target.value || 0))}
                />
              </div>
            ) : (
              <div className="space-y-2">
                <Label>Your Split (%) of shared gross</Label>
                <Input
                  className="rounded-xl"
                  type="number"
                  value={currentSplitPct}
                  min={0}
                  max={100}
                  onChange={(e) => setCurrentSplitPct(Number(e.target.value || 0))}
                />
              </div>
            )}

            <div className="space-y-2">
              <Label>Per‑File Fee (Current Employer)</Label>
              <Input
                className="rounded-xl"
                type="number"
                value={currentPerFileFee}
                min={0}
                step={50}
                onChange={(e) => setCurrentPerFileFee(Number(e.target.value || 0))}
              />
              <p className="text-xs text-muted-foreground">
                Tech/processing/admin you pay per file (optional).
              </p>
            </div>
          </CardContent>
        </Card>
      </div>

      {/* Side‑by‑side results */}
      <div className="grid md:grid-cols-2 gap-4 mt-6">
        {/* Current */}
        <Card className="rounded-2xl border-slate-200">
          <CardContent className="p-5">
            <HeaderTag label="Current" colorClass="bg-slate-100 text-slate-800" />
            <h3 className="text-lg font-semibold mt-2">Your Current Shop</h3>
            <p className="text-xs text-muted-foreground mt-1">{current.note}</p>
            <div className="mt-4 space-y-2">
              <Row label="Per‑Loan Take‑Home" value={fmtUSD(current.perLoan)} />
              <Row label="Monthly (× Loans/mo)" value={fmtUSD(current.monthly)} />
              <Row label="Annual" value={fmtUSD(current.annual)} emphasize />
            </div>
          </CardContent>
        </Card>

        {/* WeFund */}
        <Card className="rounded-2xl border-[#338e3d]/30 shadow-sm">
          <CardContent className="p-5">
            <HeaderTag
              label="WeFund"
              colorClass="bg-[#338e3d]/10 text-[#338e3d] border border-[#338e3d]/30"
            />
            <h3 className="text-lg font-semibold mt-2">WeFund — 100% minus $795</h3>
            <p className="text-xs text-muted-foreground mt-1">{wefund.note}</p>
            <div className="mt-4 space-y-2">
              <Row label="Per‑Loan Take‑Home" value={fmtUSD(wefund.perLoan)} />
              <Row label="Monthly (× Loans/mo)" value={fmtUSD(wefund.monthly)} />
              <Row label="Annual" value={fmtUSD(wefund.annual)} emphasize />
            </div>
            {/* CTA ribbon */}
            <div className="mt-4 rounded-xl border border-[#338e3d]/30 bg-[#338e3d]/5 p-3 text-sm">
              <div className="font-medium text-[#338e3d]">Why LOs pick WeFund</div>
              <ul className="mt-1 text-xs text-muted-foreground list-disc ml-5 space-y-1">
                <li>Keep 100% of your gross, simple $795/file</li>
                <li>Modern LOS + free custom CRM</li>
                <li>Broker & Correspondent access on one platform</li>
              </ul>
            </div>
          </CardContent>
        </Card>
      </div>

      {/* Delta Banner */}
      <div
        className={`mt-6 rounded-2xl p-4 border ${
          deltaAnnual >= 0
            ? "bg-[#338e3d]/5 border-[#338e3d]/30"
            : "bg-rose-50 border-rose-200"
        }`}
      >
        <div className="flex flex-wrap items-center justify-between gap-3">
          <div className="text-sm">Annual difference vs your current shop:</div>
          <div
            className={`text-xl font-semibold ${
              deltaAnnual >= 0 ? "text-[#338e3d]" : "text-rose-600"
            }`}
          >
            {deltaAnnual >= 0 ? "+" : ""}
            {fmtUSD(deltaAnnual)} / year
          </div>
        </div>
      </div>

      <p className="text-[11px] text-muted-foreground mt-4">
        Estimates only. We assume the same gross commission per loan in both scenarios. Current shop subtracts any per‑file fees you enter. WeFund model = gross − $795 per file.
      </p>
    </div>
  );
}

function HeaderTag({ label, colorClass }: { label: string; colorClass: string }) {
  return (
    <span className={`inline-flex items-center text-xs px-2 py-1 rounded-full ${colorClass}`}>
      {label}
    </span>
  );
}

function Row({ label, value, emphasize = false }: { label: string; value: string; emphasize?: boolean }) {
  return (
    <div className="flex items-center justify-between">
      <span className="text-sm text-muted-foreground">{label}</span>
      <span className={`text-base ${emphasize ? "font-semibold" : ""}`}>{value}</span>
    </div>
  );
}

/**
 * Opt-in runtime tests (WON'T RUN unless you set window.__RUN_WEFUND_TESTS__ = true in the console)
 * These validate the core math without touching the UI.
 */
// eslint-disable-next-line @typescript-eslint/ban-ts-comment
// @ts-ignore
if (typeof window !== "undefined" && (window as any).__RUN_WEFUND_TESTS__) {
  const cases: Array<{ name: string; i: ComparisonInputs; expect: Partial<ComparisonResult> }> = [
    {
      name: "Split model, basic gross in bps, no fees",
      i: {
        avgLoanAmount: 450000,
        loansPerMonth: 4,
        grossMode: "bps",
        grossBps: 200,
        grossDollar: 0,
        structure: "split",
        currentPayoutBps: 0,
        currentSplitPct: 70,
        currentPerFileFee: 0,
        wefundFlatFee: 795,
      },
      expect: {
        current: { perLoan: 450000 * (200 / 10000) * 0.7 }, // 6300
        wefund: { perLoan: 450000 * (200 / 10000) - 795 }, // 8205
      },
    },
    {
      name: "Flat BPS model with fee",
      i: {
        avgLoanAmount: 400000,
        loansPerMonth: 5,
        grossMode: "dollar",
        grossBps: 0,
        grossDollar: 8000, // same gross for WeFund
        structure: "bps",
        currentPayoutBps: 120, // 4800
        currentSplitPct: 0,
        currentPerFileFee: 250, // 4550
        wefundFlatFee: 795, // 7205
      },
      expect: {
        current: { perLoan: 400000 * (120 / 10000) - 250 },
        wefund: { perLoan: 8000 - 795 },
      },
    },
    {
      name: "Clamping: negative/over limits handled",
      i: {
        avgLoanAmount: -100, // clamp to 0
        loansPerMonth: -1, // clamp to 0
        grossMode: "bps",
        grossBps: -50, // clamp in helper path
        grossDollar: -1,
        structure: "split",
        currentPayoutBps: -10,
        currentSplitPct: 150, // clamp to 100
        currentPerFileFee: -500, // clamp in math path
        wefundFlatFee: 795,
      },
      expect: {
        current: { perLoan: 0 },
        wefund: { perLoan: 0 },
      },
    },
  ];

  cases.forEach(({ name, i, expect }) => {
    const r = computeComparison(i);
    const approx = (a: number | undefined, b: number | undefined) =>
      typeof a === "number" && typeof b === "number" && Math.abs(a - b) < 1e-6;

    if (expect.current?.perLoan !== undefined && !approx(r.current.perLoan, expect.current.perLoan)) {
      console.error(`[TEST FAIL] ${name}: current.perLoan`, r.current.perLoan, "!=", expect.current.perLoan);
    }
    if (expect.wefund?.perLoan !== undefined && !approx(r.wefund.perLoan, expect.wefund.perLoan)) {
      console.error(`[TEST FAIL] ${name}: wefund.perLoan`, r.wefund.perLoan, "!=", expect.wefund.perLoan);
    }
  });

  console.log("WeFund calculator tests executed. Set window.__RUN_WEFUND_TESTS__ = true to rerun.");
}
