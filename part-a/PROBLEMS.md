# IRCTC Problem Discovery — Part A

## Summary
- Total problems documented: 6 (3 given + 3 self-discovered)
- Platform explored: irctc.co.in (live, as of June 2026)
- Devices used: Desktop Chrome, Mobile Chrome

---

## Problem 1: Tatkal Booking Crashes at 10:00 AM [Given]

**What is broken:**
At exactly 10:00 AM when Tatkal booking opens, the IRCTC server becomes unresponsive. Users get blank screens, spinning loaders, or session timeout errors with no explanation. There is no queue system, no position indicator, and no feedback on whether the request is being processed. Users who prepared for 30 minutes lose their chance in seconds with no actionable information.

**Affected users:**
Millions of daily commuters, emergency travelers, and people booking last-minute travel. Estimated 8 crore registered users, with lakhs attempting Tatkal simultaneously at 10:00 AM every single day.

**Frequency:**
Daily — every morning at 10:00 AM without exception. Worse on Mondays and days before long weekends. A known, recurring, unresolved issue.

**Current flow — step by step:**
1. User opens irctc.co.in at 9:50 AM and logs in
2. User searches for train between source and destination
3. User selects date and clicks on Tatkal quota
4. System shows available seats at 9:58 AM
5. User selects train and clicks Book Now at 9:59 AM
6. Passenger details page loads — user fills in details quickly
7. At 10:00 AM user clicks Proceed to Payment
8. Page spins indefinitely — no response from server
9. After 60 seconds, session times out or throws generic error
10. User refreshes — Tatkal quota now shows "Not Available"
11. User has lost the booking with no explanation or retry option

**Where exactly it breaks:**
Step 7-8: The payment gateway handoff at exactly 10:00 AM when server load spikes from lakhs of simultaneous requests. No queue, no rate limiting visible to user, no feedback on failure reason.

---

## Problem 2: Search Filters Do Not Work Reliably [Given]

**What is broken:**
When users apply filters on train search results (class type, departure time, availability), the results either do not update, update incorrectly, or reset when the user navigates back. Filters do not persist across page interactions, forcing users to reapply them repeatedly.

**Affected users:**
All users searching for trains — particularly senior citizens and first-time users who rely on filters to narrow down results. Affects every user who searches trains, estimated lakhs daily.

**Frequency:**
Every search session. The filter reset on back-navigation happens 100% of the time. Incorrect filtering occurs frequently depending on network conditions.

**Current flow — step by step:**
1. User opens irctc.co.in and navigates to Train Search
2. User enters source, destination, and date
3. Search results load showing 15-20 trains
4. User clicks on "Sleeper" class filter to narrow results
5. Page appears to reload but shows same results as before
6. User applies "Available" filter to show only bookable trains
7. Some unavailable trains still appear in results
8. User clicks on a train to check details
9. User clicks browser back button
10. All filters have reset — full unfiltered list shown again
11. User must reapply all filters from scratch

**Where exactly it breaks:**
Step 5 and Step 10: Filter state is not maintained in application memory or URL parameters. Each filter change triggers an inconsistent server-side query, and no filter state is preserved on back navigation.

---

## Problem 3: Seat Selection Resets [Given]

**What is broken:**
During the booking flow, when a user selects a specific berth (lower, middle, upper) on the seat map and proceeds to the passenger details page, the selected berth is not carried forward. The system either shows a different berth or reverts to auto-assignment. On mobile this happens nearly every attempt.

**Affected users:**
Senior citizens who require lower berths, passengers with medical needs, families traveling together who need adjacent seats. Particularly severe for users who specifically need lower berths for health reasons.

**Frequency:**
Occurs frequently on mobile — estimated 60-70% of seat selection attempts on mobile browser result in reset. Desktop reset rate lower but still present under load.

**Current flow — step by step:**
1. User completes train search and selects a train
2. User selects class (e.g. Sleeper) and clicks Book Now
3. Seat map loads showing coach layout with berth numbers
4. User taps on berth 23 (Lower) — it highlights as selected
5. User clicks Proceed button
6. Passenger details page loads
7. Berth preference field shows "No Preference" or a different berth
8. User goes back to seat map to reselect
9. Seat map reloads with no previous selection remembered
10. User selects again and proceeds
11. Same reset occurs — berth preference lost again

**Where exactly it breaks:**
Step 6-7: Selected berth data is not passed in the session state between the seat map screen and passenger details screen. The front-end selection is not bound to the booking payload sent to the server.

---

## Problem 4: No Real-Time PNR Status Updates [Self-Discovered]

**How I found it:**
Navigated to the PNR Status page on irctc.co.in. Entered a PNR number and checked the status. Noticed the page shows a static "last updated" timestamp with no auto-refresh. Users must manually reload the page repeatedly to check for chart preparation or boarding platform updates.

**What is broken:**
The PNR status page is completely static. There is no live update, no push notification, no auto-refresh. Users waiting for chart preparation (which determines confirmed vs RAC vs waitlist) must manually reload the page every few minutes. The page also does not show platform number even when it is available at the station.

**Affected users:**
Every passenger checking PNR status — particularly waitlisted passengers anxiously waiting for confirmation, and passengers at stations trying to find their platform. Estimated lakhs of PNR checks daily.

**Frequency:**
Every single PNR status check session. The static nature of the page is a permanent design flaw, not an intermittent bug.

**Current flow — step by step:**
1. User opens irctc.co.in and clicks PNR Status
2. User enters 10-digit PNR number
3. Page loads showing current status — e.g. "WL 45"
4. User wants to know if status has changed to confirmed
5. User waits 5 minutes and manually reloads the page
6. Page shows same static data with old timestamp
7. User cannot tell if data is fresh or cached
8. User reloads again 10 minutes later
9. No indication of when chart will be prepared
10. User misses confirmation update and boards wrong coach

**Where exactly it breaks:**
Step 3-6: The page renders server-side static HTML with no WebSocket or polling mechanism. Data is fetched once at page load and never updated without a full manual reload.

---

## Problem 5: Mobile Login Session Expires Too Frequently [Self-Discovered]

**How I found it:**
Opened irctc.co.in on mobile Chrome browser. Logged in successfully. Switched to another app for 3 minutes to check train time. Returned to IRCTC. Was shown the login page again with session expired message — had to log in again from scratch including captcha.

**What is broken:**
IRCTC mobile browser session timeout is extremely short — approximately 2-3 minutes of inactivity. Users who switch apps briefly to check messages, UPI apps for payment reference, or calendar for travel dates are logged out and forced to restart the entire booking flow. The captcha on re-login adds additional friction.

**Affected users:**
All mobile browser users — which is a significant portion of India's internet users who primarily use mobile. Particularly affects users in the middle of booking who need to reference another app.

**Frequency:**
Every mobile session where the user switches apps for more than 2-3 minutes. Extremely common behavior during booking when users check UPI apps or messaging apps.

**Current flow — step by step:**
1. User opens irctc.co.in on mobile Chrome
2. User logs in with username, password, and captcha
3. User searches for train and selects it
4. User switches to WhatsApp to confirm travel dates with family
5. User spends 3 minutes in WhatsApp
6. User returns to IRCTC browser tab
7. Page shows "Your session has expired, please login again"
8. User must solve captcha again and re-enter credentials
9. User's previous search and selection is completely lost
10. User must restart the entire booking flow from scratch

**Where exactly it breaks:**
Step 6-7: Session token TTL is set too short for mobile use cases. No session persistence or background token refresh mechanism exists. No warning is shown before session expiry.

---

## Problem 6: Cancellation and Refund Flow Is Confusing and Opaque [Self-Discovered]

**How I found it:**
Navigated to the cancellation section under booked tickets. Tried to understand the cancellation charges before confirming cancellation. Found no clear breakdown of what refund amount would be received before clicking the final confirm button.

**What is broken:**
The cancellation flow does not show the user the exact refund amount before they confirm cancellation. Users must cancel first and then check the refund amount. The TDR (Ticket Deposit Receipt) filing process is buried under multiple menus with no clear guidance. Cancellation charges policy is shown as a generic table elsewhere on the site — not contextually during the cancellation flow.

**Affected users:**
Any passenger who needs to cancel a ticket — due to plan changes, emergencies, or booking errors. Particularly affects users who are unsure whether cancellation is worth it financially and need to see the refund amount to decide.

**Frequency:**
Every cancellation attempt. The missing refund preview is a permanent design gap in the cancellation flow.

**Current flow — step by step:**
1. User logs in and goes to Booked Ticket History
2. User finds the ticket they want to cancel
3. User clicks Cancel Ticket
4. A confirmation screen appears asking "Are you sure?"
5. User looks for refund amount to make an informed decision
6. No refund amount is shown on this screen
7. User searches the page — finds only a generic link to cancellation policy
8. User must open a new tab to find the charges table
9. User manually calculates the refund based on the table
10. User returns and clicks Confirm Cancellation
11. System processes cancellation and shows refund amount — only after it is already done

**Where exactly it breaks:**
Step 5-6: The backend has all data needed to calculate the exact refund (ticket price, cancellation time, journey date, class) but does not surface this calculation to the user before the irreversible confirmation step.