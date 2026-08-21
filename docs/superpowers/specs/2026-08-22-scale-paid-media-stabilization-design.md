# Scale Paid Media Lead-Generation Stabilization Design

## Problem
Scale Studio currently lets the AgencyAnalytics PDF parser, legacy report state (july, week31, benchmarkWindow, latestWeek), global generated-report week validation, and the Strategic Shift card depend on the same strings. The August 1–19 AgencyAnalytics PDF legitimately contains different current weeks by segment: Core is Week 34 while Named is Week 33. It also uses This Month instead of an explicit date range for the benchmark window.

## Goal
Stabilize Paid Media import so the report consumes normalized segment-specific lead-generation values, the Named Strategic Shift card never falls back to zero when supported source values exist, and Build / Refresh Preview is not blocked by cross-segment week-label validation.

## Architecture
Introduce a canonical leadGen snapshot per segment between parseAgencyAnalyticsPaidPdf() and report rendering. Preferred benchmark source is the Lead Generation narrative. Named falls back to Named Meta Summary Metrics when the narrative benchmark is absent. Core and Named normalize independently. Legacy fields remain for backward compatibility.

## Scope Constraints
- Do not change Preview renderer behavior.
- Do not change private live-report links or publishing flow.
- Do not change PDF export.
- Do not change workspace restore behavior.
- Do not change creative-library logic.
- Preserve automatic PDF fallback and page-independent PDF parsing.
- Preserve legacy fields for backward compatibility.

## Acceptance Criteria
1. Named benchmark = 18 leads / $267 CPL.
2. Named latest week = Week 33 / 8 leads / $157 CPL.
3. Core benchmark = 172 leads / $59 CPL.
4. Core latest week = Week 34 / 30 leads / $71.49 CPL.
5. Strategic Shift renders Named values instead of zero.
6. Core Week 34 and Named Week 33 coexist without build validation error.
7. Existing Preview renderer remains unchanged.
8. Named summary metrics provide benchmark fallback if narrative benchmark is absent.
