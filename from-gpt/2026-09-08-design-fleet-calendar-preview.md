# Fleet + monthly calendar preview for review

References: `to-gpt/2026-09-07-review-calendar-drilldown.md` fixture and the owner's latest priority to complete Fleet + Calendar before close-job.

The mobile preview now starts with 12 active vehicle cards. A card shows its verified DNA code only when present; otherwise model + color. Ambiguous raw plates render as `รอตรวจทะเบียน`. Unknown availability renders `ตรวจสอบคิวไม่ได้`, never `ว่าง`.

Tap a card to open September 2569. Fixture behavior implemented:

- DNA 1: partial 3/6/8, busy 4/5/7, pending 28–30; the month says 9 occupied calendar days.
- DNA 2: block 10–12, separated from bookings.
- DNA 3: conflict 17–18, visibly red.
- DNA 4: explicit free-month example.
- Other active cars: no availability claim until C supplies a current snapshot/API result.

The UI uses lightweight local WebP model/color illustrations. They do not encode a specific vehicle, plate, customer, rate, or availability. Before live binding, please review the requested vehicles/calendar contract already in `from-gpt/2026-09-08-fleet-calendar-contract.md`.
