# Scale Paid Media Stabilization Implementation Plan

Goal: decouple Paid Media lead-generation parsing from legacy week labels.

1. Add canonical paidLeadGenPoint/buildPaidLeadGenSnapshot helpers.
2. Populate paid.core.leadGen and paid.named.leadGen during PDF apply while preserving legacy fields.
3. Make buildReport use canonical leadGen state with legacy fallback.
4. Remove only the invalid global cross-segment week scan from validateGeneratedReport.
5. Verify exact August 1–19 values and preservation markers before integration.
