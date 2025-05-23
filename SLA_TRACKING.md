# SLA (Service Level Agreement) Tracking Mechanism

This document describes the mechanism for defining, monitoring, and reporting on Service Level Agreements (SLAs) within the IT helpdesk system. The goal is to ensure timely responses and resolutions for user issues.

## 1. SLA Policy Definition

Administrators will define SLA policies through a dedicated section in the admin panel. Each policy specifies targets for response and resolution times based on various ticket criteria.

*   **Admin Interface:**
    *   A user-friendly interface will allow admins to create, edit, delete, and prioritize SLA policies.

*   **Policy Conditions (Criteria for Application):**
    *   Policies can be triggered based on a combination of:
        *   **Ticket Priority:** (e.g., Critical, High, Medium, Low). Different priorities will have different time targets.
        *   **Ticket Category/Sub-Category:** (e.g., Hardware > Laptop, Software > Application X, Network > VPN).
        *   **User Group/Organization:** Specific SLAs for VIP users, particular departments, or client companies.
        *   **Ticket Type:** (e.g., Incident, Service Request, Problem).
    *   A **default/fallback policy** will apply if no other policy conditions are met.
    *   **Policy Order/Priority:** Admins can define the order in which policies are evaluated (e.g., the most specific matching policy wins, or a numerical priority).

*   **Configurable Parameters for each Policy:**
    *   **Name and Description:** For easy identification.
    *   **Time to First Response (TTR):** The maximum time allowed for an agent to provide the first meaningful (public) response to the user after ticket creation.
    *   **Time to Resolution (TTR):** The maximum time allowed for the ticket to be marked as resolved/closed from the moment of creation (or from when it's no longer paused).
    *   **Operational Hours (Business Hours):**
        *   Definable working days (e.g., Monday-Friday).
        *   Definable working hours per day (e.g., 9:00 AM - 5:00 PM).
        *   Holiday Calendar: A list of non-working days. SLA timers will only run during these defined operational hours.
    *   **Escalation Rules:** (See Section 4) Who to notify and what actions to take at different stages (warning, breach).
    *   **Pause Conditions:** Which ticket statuses (e.g., "Pending User Input", "On Hold - Awaiting Vendor") should pause the SLA clock.

## 2. SLA Monitoring

The system will actively monitor ticket lifecycles against the applicable SLA policies. This is primarily the responsibility of the **SLA Service**.

*   **SLA Service Role:**
    *   The SLA Service subscribes to events published by the **Ticketing Service** via the **Message Queue**.
    *   Key events include:
        *   `ticket_created`: Triggers SLA policy matching and starts the initial "Time to First Response" and "Time to Resolution" clocks.
        *   `agent_first_responded`: Marks the achievement of the "Time to First Response" SLA. This is typically when an agent posts the first public comment/reply.
        *   `ticket_status_changed`: Critical for pausing or resuming SLA clocks based on the new status (e.g., "Pending User Input" pauses, "Open" resumes).
        *   `ticket_resolved` / `ticket_closed`: Marks the achievement of the "Time to Resolution" SLA.
        *   `ticket_priority_changed`, `ticket_category_changed`, etc.: May trigger a re-evaluation of the applied SLA policy and recalculation of target times.
*   **SLA Instance Management:**
    *   When a ticket is created, the SLA Service determines the applicable SLA policy based on the ticket's attributes.
    *   It then creates an "SLA instance" for that ticket, calculating and storing the specific due dates/times for first response and resolution, taking into account the policy's operational hours.
    *   This instance tracks the current state of the SLA (e.g., active, paused, breached).

## 3. Timer Management

Accurate timer management is crucial for SLA tracking.

*   **Calculation Logic:**
    *   SLA timers (for both response and resolution) only "tick" during the operational hours defined in the applied SLA policy.
    *   Example: A ticket with a 4-hour resolution SLA is created 1 hour before the end of the business day. The clock runs for 1 hour, pauses overnight, and resumes the next business day with 3 hours remaining.
*   **Pausing Timers:**
    *   When a ticket is moved to a status defined in the SLA policy as a "pause" state (e.g., "Pending User Response", "On Hold - Awaiting Parts"), the SLA Service pauses both the response and resolution timers.
    *   The duration for which the ticket remains in a paused state does not count against the SLA.
*   **Resuming Timers:**
    *   When the condition for the pause is lifted (e.g., the user replies to a "Pending User Response" ticket, or the ticket is moved back to an "Open" or "In Progress" status), the SLA Service resumes the timers.
*   **Recalculation:**
    *   If a ticket's attributes change (e.g., priority is escalated) such that a different SLA policy applies, or the current policy has different targets for the new attribute:
        *   The SLA Service will re-evaluate the applicable policy.
        *   Target due dates/times will be recalculated. This might involve considering the time already elapsed under the previous policy conditions.

## 4. Notifications and Escalations

The system will provide timely notifications for impending and actual SLA breaches to prompt action.

*   **Warning Notifications (Pre-Breach):**
    *   Configurable thresholds (e.g., "warn if 75% of SLA time has elapsed" or "warn 1 hour before breach").
    *   Notifications sent to the assigned agent and/or their manager.
*   **Breach Notifications:**
    *   Immediate notifications when a "Time to First Response" or "Time to Resolution" SLA is breached.
    *   Sent to assigned agent, manager, and potentially other designated stakeholders (e.g., SLA manager, department head).
*   **Notification Channels:**
    *   **In-App:** Visual indicators on tickets (e.g., color-coding: green for on-track, amber for warning, red for breached), dashboard notifications, and prioritized lists for agents.
    *   **Email:** Customizable email templates for warnings and breaches.
    *   **Message Queue Events:** The SLA Service can publish events like `sla_target_approaching`, `sla_target_breached`, `sla_escalated` to the message queue. Other services could subscribe to these for custom workflows (e.g., advanced alerting, integration with external systems).
    *   **Future Integrations:** Webhooks for Slack, Microsoft Teams, PagerDuty, etc.
*   **Automated Escalation Rules (Optional, configurable per policy):**
    *   **Priority Increase:** Automatically escalate the ticket's priority upon SLA breach or repeated warnings.
    *   **Reassignment:** Automatically reassign the ticket to a predefined escalation queue, senior agent, or manager.
    *   **Management Alerts:** Notify higher levels of management if a breach is not addressed within a certain timeframe after the initial breach notification.

## 5. SLA Reporting

Comprehensive reporting on SLA performance is vital for assessing helpdesk effectiveness and identifying areas for improvement.

*   **Key Performance Indicators (KPIs):**
    *   **Overall SLA Compliance:** Percentage of tickets that met all applicable SLA targets (both first response and resolution).
    *   **First Response SLA Compliance Rate:** Percentage of tickets meeting the TTR target.
    *   **Resolution SLA Compliance Rate:** Percentage of tickets meeting the resolution time target.
    *   **Breach Rate (First Response & Resolution):** Percentage of tickets that breached these targets.
    *   **Average Time to First Response.**
    *   **Average Time to Resolution.**
    *   **Number of Breaches by Policy/Priority/Category.**
    *   **List of Breached Tickets:** Details of tickets that failed to meet SLAs.
*   **Report Dimensions & Filters:**
    *   Ability to filter reports by date range, ticket priority, category, type, assigned agent/team, user group.
    *   Trend analysis (e.g., SLA performance over weeks/months).
    *   Comparisons between different agents, teams, or ticket categories.
*   **Report Formats:**
    *   Dashboards with visual charts and graphs for quick overviews.
    *   Tabular reports for detailed analysis, exportable to CSV/Excel.
*   **Target Audience:**
    *   **Agents:** To track their individual performance.
    *   **Team Leaders/Managers:** To monitor team performance, identify bottlenecks, and manage workload.
    *   **Helpdesk Management:** To assess overall operational efficiency, justify resource allocation, and demonstrate value.

This SLA tracking mechanism aims to provide a robust framework for ensuring service quality and responsiveness within the IT helpdesk system. Regular review and refinement of SLA policies and performance will be essential.
