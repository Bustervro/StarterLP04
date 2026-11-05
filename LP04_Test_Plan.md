# Learning Practice 4: Black-Box Testing, Part 2

## 1. Application and Approach

**Application:** North Vacation Booking  
**Features:** Reservation validation and pricing; reservation status changes  
**Method:** Black-box test design using only the supplied requirements, without access to application source code.

**Important:** This is a set of **designed test cases and expected results**, not a report of tests run against a working application. For all cases marked **Available**, assume every night requested is open. Dates are illustrative and are set in 2027.

## 2. Requirements and Pricing Formula

A reservation is accepted only if the checkout date is after the check-in date, it lasts **1–31 nights**, it includes **1–12 guests**, and **all requested nights are available**. Checkout day is not a reserved night.

For valid and available reservations:

```
Nights = checkout date - check-in date
Base cost = nights × $200
Cleaning fee = $50 (once per reservation)
Extra guest fee = max(guests - 5, 0) × $25 × nights
Total = base cost + cleaning fee + extra guest fee
```

Invalid reservation information raises `ValueError`. Unavailable dates raise `UnavailableError`. A newly accepted reservation starts in `Pending`.

## 3. Step 1 — Five Conditions and Five Actions

### Five conditions

| Condition | Meaning |
|---|---|
| **C1** | Check-in and checkout dates are valid, with check-in before checkout. |
| **C2** | Requested number of nights is between 1 and 31, inclusive. |
| **C3** | Requested number of guests is between 1 and 12, inclusive. |
| **C4** | Every requested night is available. |
| **C5** | There are more than five guests. |

### Five actions

| Action | Meaning |
|---|---|
| **A1** | Create the reservation, initially in `Pending`. |
| **A2** | Calculate the standard price: `$200 × nights + $50` cleaning fee. |
| **A3** | Add `$25 × nights` for each guest beyond five. |
| **A4** | Raise `ValueError` for invalid information. |
| **A5** | Raise `UnavailableError` because at least one requested night is booked. |

## 4. Step 2 — Decision Table

`T` = true; `F` = false; `-` = not relevant; `X` = expected action.

| Conditions and actions | R1 | R2 | R3 | R4 | R5 | R6 | R7 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **C1** Dates valid and in order? | T | T | F | T | T | T | T |
| **C2** Nights within 1–31? | T | T | - | F | T | T | T |
| **C3** Guests within 1–12? | T | T | T | T | F | F | T |
| **C4** Every night available? | T | T | - | - | - | - | F |
| **C5** More than five guests? | F | T | - | - | F | T | - |
| **A1** Create `Pending` reservation | X | X | | | | | |
| **A2** Calculate standard price | X | X | | | | | |
| **A3** Add extra guest fee | | X | | | | | |
| **A4** Raise `ValueError` | | | X | X | X | X | |
| **A5** Raise `UnavailableError` | | | | | | | X |

**What each rule means:**

- **R1:** Valid, available reservation for 1–5 guests; charge the standard nightly cost and one-time cleaning fee.
- **R2:** Valid, available reservation for 6–12 guests; also charge the extra guest fee.
- **R3:** The dates are invalid or not ordered correctly; reject with `ValueError`.
- **R4:** The dates are ordered but the stay exceeds 31 nights; reject with `ValueError`.
- **R5:** Too few guests (zero); reject with `ValueError`.
- **R6:** Too many guests (13); reject with `ValueError`.
- **R7:** Reservation information is otherwise valid, but a requested night is unavailable; reject with `UnavailableError`.

**Assumption:** Each invalid test changes one requirement at a time. The requirements do not define which error wins when multiple inputs are wrong, so this table does not assume an error-checking order.

## 5. Step 3 — Decision Table Test Cases

All dates are in **2027**. "Available" means *every requested night is available*; "Unavailable" means *at least one requested night is not available*. Each test starts as a new reservation request.

| Test ID | Rule | Conditions | Check-in | Checkout | Guests | Availability | Expected action | Expected price or exception |
|---|---|---|---|---|---:|---|---|---|
| **DT-01** | R1 | Valid dates, 3 nights, guest count valid, no extra guests | Jan 10 | Jan 13 | 3 | Available | Create reservation in `Pending`; standard pricing | **$650** (`3 × $200 + $50`) |
| **DT-02** | R2 | Valid dates, 3 nights, 3 extra guests | Jan 10 | Jan 13 | 8 | Available | Create reservation in `Pending`; standard pricing plus extra guest fee | **$875** (`$600 + $50 + 3 × $25 × 3`) |
| **DT-03** | R3 | Checkout is before check-in | Jan 13 | Jan 12 | 3 | Available | Reject request | **`ValueError`** |
| **DT-04** | R4 | Valid date order, but stay is 32 nights | Jan 1 | Feb 2 | 3 | Available | Reject request | **`ValueError`** |
| **DT-05** | R5 | Guest count is below 1 | Jan 10 | Jan 13 | 0 | Available | Reject request | **`ValueError`** |
| **DT-06** | R6 | Guest count is above 12 | Jan 10 | Jan 13 | 13 | Available | Reject request | **`ValueError`** |
| **DT-07** | R7 | Valid 3-night stay, but Jan 11 is unavailable | Jan 10 | Jan 13 | 3 | Unavailable | Reject request | **`UnavailableError`** |
| **DT-08** | R3 | Check-in equals checkout (zero nights) | Jan 10 | Jan 10 | 3 | Available | Reject request | **`ValueError`** |
| **DT-09** | R1 | Lower boundaries: 1 night, 1 guest | Jan 10 | Jan 11 | 1 | Available | Create `Pending` reservation; standard price | **$250** (`$200 + $50`) |
| **DT-10** | R1 | Upper night boundary: 31 nights, 5 guests | Jan 1 | Feb 1 | 5 | Available | Create `Pending` reservation; no extra guest fee | **$6,250** (`31 × $200 + $50`) |
| **DT-11** | R2 | Upper guest boundary: 12 guests, 1 night | Jan 10 | Jan 11 | 12 | Available | Create `Pending` reservation; 7 extra guests | **$425** (`$200 + $50 + 7 × $25`) |

**How to evaluate each decision-table test:** Supply the dates, guest count, and availability shown. Compare the resulting reservation state and total price—or exception—with the expected result. Do not charge a cleaning fee for a rejected request.

## 6. Step 4 — Reservation States and Events

### Five states

| State | Description |
|---|---|
| **Pending** | New valid reservation, awaiting payment result or cancellation. |
| **Confirmed** | Successful payment has confirmed the reservation. |
| **Checked In** | The guest has checked in on the scheduled check-in date. |
| **Completed** | The guest has checked out and the reservation is finished. |
| **Cancelled** | The reservation was cancelled or its payment failed. |

### Five events

| Event | Meaning |
|---|---|
| **E1 — Payment succeeds** | A pending reservation is successfully paid. |
| **E2 — Payment fails** | A pending payment attempt fails. |
| **E3 — Guest cancels** | A guest cancels an eligible reservation. |
| **E4 — Guest checks in** | The guest tries to check in. |
| **E5 — Guest checks out** | The guest checks out after the stay. |

`Cancelled` and `Completed` are final states: they cannot change to another state. Any invalid transition raises `InvalidStateError` and leaves the reservation in its current state.

## 7. Step 5 — State Transition Table

| Transition ID | Current state | Event / condition | Expected state | Expected result | Type |
|---|---|---|---|---|---|
| **TR-01** | Pending | E1: Payment succeeds | Confirmed | Confirm reservation | Valid |
| **TR-02** | Pending | E2: Payment fails | Cancelled | Cancel reservation | Valid |
| **TR-03** | Pending | E3: Guest cancels | Cancelled | Cancel pending reservation | Valid |
| **TR-04** | Confirmed | E4: Guest checks in **on check-in date** | Checked In | Check guest in | Valid |
| **TR-05** | Confirmed | E3: Guest cancels | Cancelled | Cancel confirmed reservation | Valid |
| **TR-06** | Checked In | E5: Guest checks out **after stay** | Completed | Complete reservation | Valid |
| **TR-07** | Pending | E4: Guest tries to check in | Pending | Raise `InvalidStateError`; no state change | Invalid |
| **TR-08** | Confirmed | E5: Guest tries to check out before checking in | Confirmed | Raise `InvalidStateError`; no state change | Invalid |
| **TR-09** | Checked In | E3: Guest tries to cancel | Checked In | Raise `InvalidStateError`; no state change | Invalid |
| **TR-10** | Cancelled | E1: Payment succeeds | Cancelled | Raise `InvalidStateError`; no state change | Invalid |
| **TR-11** | Completed | E3: Guest tries to cancel | Completed | Raise `InvalidStateError`; no state change | Invalid |
| **TR-12** | Confirmed | E4: Guest tries to check in **before check-in date** | Confirmed | Raise `InvalidStateError`; no state change | Invalid |

**Coverage:** 6 valid transitions plus 6 invalid transitions = **12 transitions**, exceeding the minimum of nine.

## 8. Step 6 — State Transition Test Cases

**Common setup:** Create a valid, available reservation for **January 10–13, 2027**, with **3 guests**, expected total **$650**. It begins in `Pending`. To reach a later starting state, perform the earlier valid transitions first. For check-in tests, use January 10, 2027; for completed checkout, assume the guest checks out after the stay on January 13, 2027. Reset to a new reservation for each independent test.

| Test ID | Starting state | Test action / setup | Expected ending state | Expected result | Type |
|---|---|---|---|---|---|
| **ST-01** | Pending | Process successful payment | Confirmed | Reservation changes to `Confirmed` | Valid (TR-01) |
| **ST-02** | Pending | Process failed payment | Cancelled | Reservation changes to `Cancelled` | Valid (TR-02) |
| **ST-03** | Pending | Guest requests cancellation | Cancelled | Pending reservation is cancelled | Valid (TR-03) |
| **ST-04** | Confirmed | On Jan 10, guest checks in | Checked In | Guest successfully checked in | Valid (TR-04) |
| **ST-05** | Confirmed | Guest requests cancellation before check-in | Cancelled | Confirmed reservation is cancelled | Valid (TR-05) |
| **ST-06** | Checked In | After stay, guest checks out on Jan 13 | Completed | Reservation changes to `Completed` | Valid (TR-06) |
| **ST-07** | Pending | Guest attempts check-in before confirmation | Pending | `InvalidStateError`; no state change | Invalid (TR-07) |
| **ST-08** | Confirmed | Guest attempts checkout without checking in | Confirmed | `InvalidStateError`; no state change | Invalid (TR-08) |
| **ST-09** | Checked In | Guest attempts cancellation | Checked In | `InvalidStateError`; no state change | Invalid (TR-09) |
| **ST-10** | Cancelled | Try to accept a successful payment | Cancelled | `InvalidStateError`; no state change | Invalid (TR-10) |
| **ST-11** | Completed | Guest attempts cancellation | Completed | `InvalidStateError`; no state change | Invalid (TR-11) |
| **ST-12** | Confirmed | Guest attempts check-in on Jan 9 (one day early) | Confirmed | `InvalidStateError`; no state change | Invalid (TR-12) |
| **ST-13** | Pending | **Complete sequence:** payment succeeds → check in on Jan 10 → check out after stay on Jan 13 | Completed | `Pending → Confirmed → Checked In → Completed` | Full sequence |
| **ST-14** | Pending | **Cancellation sequence:** payment succeeds → guest cancels before check-in | Cancelled | `Pending → Confirmed → Cancelled` | Cancellation sequence |
| **ST-15** | Pending | **Failed-payment cancellation sequence:** payment fails → guest tries to check in | Cancelled | `Pending → Cancelled`; subsequent check-in raises `InvalidStateError` | Cancellation + invalid |

**Coverage:** 6 individual valid transitions, 6 individual invalid transitions, a complete reservation sequence, and two cancellation sequences = **15 test cases** (minimum required: 11).

## 9. Summary and Submission

This test plan uses only the North Vacation Booking requirements to design black-box tests. The decision table covers normal pricing, the extra guest charge, invalid dates, invalid stay length, invalid guest counts, and unavailable dates. The state transition table covers valid reservation progress and invalid attempts to leave states that cannot be changed.

| Required work | Included |
|---|---|
| Identify five decision conditions and five possible actions | Yes |
| Decision table with at least six rules | **7 rules** |
| At least one decision-table test for every rule | **11 test cases** |
| Identify five states and five events | Yes |
| State table with six valid and three invalid transitions | **6 valid + 6 invalid** |
| At least 11 state-transition tests, including full and cancellation sequences | **15 test cases** |

**To submit:** Add this Markdown file to a **public** GitHub repository (for example, `StarterLP04`), commit it, and submit that repository URL to Canvas. The requirements do not provide an executable application, so these are expected results rather than observed test executions.

**Source:** `StarterLP04/Readme.md` provided with Learning Practice 4.
