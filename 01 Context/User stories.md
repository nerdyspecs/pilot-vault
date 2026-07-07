---
type: context
updated: 2026-07-04
---
# User Stories
Who uses this app, what they need, and what they don't care about.

---

## Workshop Side — Internal Users

### Workshop Manager

**Who are they?**
- Oversees day-to-day workshop operations
- Responsible for workshop performance, delivery timelines, and team coordination

**Goals**
- Maintain visibility of all active jobs
- Identify bottlenecks before customers complain
- Ensure resources are utilized effectively
- Keep the workshop running smoothly without constant manual follow-up

**Frustrations**
- Has to walk the workshop floor to understand what is happening
- Learns about delayed jobs too late
- Relies on verbal updates from different departments
- Lacks a clear view of workshop workload and health

---

### Front Desk / Service Advisor

**Who are they?**
- First point of contact for customers
- Responsible for job intake, communication, and coordination

**Goals**
- Register vehicles quickly and accurately
- Keep customers informed throughout the repair process
- Know the status of every active job
- Coordinate work between customers, technicians, and support staff

**Frustrations**
- Loses visibility once a vehicle enters the workshop
- Has difficulty answering customer questions confidently
- Does not always know who is working on which job
- Customer complaints and technical findings are not always communicated clearly

---

### Mechanic / Technician

**Who are they?**
- Performs inspections, diagnostics, repairs, and testing
- Responsible for completing assigned work safely and correctly

**Goals**
- Know exactly what work requires attention
- Receive complete information before starting a job
- Focus on repairs instead of chasing information
- Develop technical skills and grow professionally

**Frustrations**
- Assigned work is not always communicated clearly
- Customer complaints may be incomplete or inaccurate
- Often unaware of new work requiring attention
- Has little visibility into skill progression and career development

---

### Warehouse / Parts Staff

**Who are they?**
- Sources, orders, and tracks parts required for repairs
- Coordinates with suppliers and workshop staff

**Goals**
- Procure the correct parts quickly
- Track outstanding orders effectively
- Minimize delays caused by parts shortages
- Build knowledge of suppliers and parts history

**Frustrations**
- Has difficulty tracking which jobs are waiting on parts
- Must repeatedly follow up with suppliers manually
- Historical ordering knowledge is not easily accessible
- Similar part numbers and supplier options create confusion

---

### Workshop Supervisor / Foreman

**Who are they?**
- Leads technicians on the workshop floor
- Responsible for work allocation, quality, and technical coordination

**Goals**
- Assign the right job to the right technician
- Balance workload across the team
- Identify blocked or high-risk jobs early
- Maintain repair quality and efficiency

**Frustrations**
- Limited visibility into technician workload
- Relies heavily on personal experience when assigning work
- Difficult to monitor multiple ongoing jobs simultaneously
- Knowledge transfer between technicians is inconsistent

---

## Customer Side — External Users

### Vehicle Owner

**Who are they?**
- Owns or manages the vehicle being repaired
- Depends on the workshop to provide reliable service and updates

**Goals**
- Understand the current status of the vehicle
- Know when the vehicle will be ready
- Receive clear and transparent communication
- Minimize disruption to daily operations

**Frustrations**
- Has to call repeatedly for updates
- Uncertain about repair timelines
- Feels disconnected from the repair process
- Often receives information only when delays occur
---

## Persona → Employment role mapping
These personas are **floor research**, not the app's role enum — several map many-to-one, and
"Foreman" has no dedicated role in v1. The actual permission roles live on **Employment**
([[Design laws]] #1); see [[Data model]] and [[M1-F1 Status flow and transitions]].

| Persona (this doc) | Employment role (v1) |
|---|---|
| Workshop Manager | `workshop_manager` |
| Front Desk / Service Advisor | `service_advisor` |
| Mechanic / Technician | `technician` |
| Warehouse / Parts Staff | `parts_advisor` |
| Workshop Supervisor / Foreman | no dedicated role — capability folded into `workshop_manager` |
| Vehicle Owner | not an Employment role — read-only via token link, no login ([[Design laws]] #8) |

## Design constraints from users
- Mechanics and warehouse staff need minimal-click interfaces — they work on the floor, not at a desk
- Front desk needs speed — intake should take under a minute
- Owner-facing view must work on mobile without an app install
- The system must work even if not all staff adopt it perfectly — partial use should still provide value
