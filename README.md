# System Design Portfolio

A collection of system designs I created for real-world projects, covering architecture decisions, trade-offs, and scalability considerations. Each write-up documents the requirements, architecture, data model, critical design decisions, failure modes, and paths for scaling the system.

## Designs

| Design | Summary | Key challenge |
| --- | --- | --- |
| [Doctor Appointment Booking System](doctor-appointment-booking/) | Search for practitioners and manage appointment reservations, bookings, cancellations, and rescheduling. | Combining eventually consistent search with strongly consistent booking to prevent double reservations. |
| [Stocks Broker](stock-broker/) | Stream live stock prices and place, track, and cancel orders routed to an external exchange. | Combining a highly available live price stream with strongly consistent orders: one dispatch per order across instances, and reconciliation with the exchange. |

<!-- Add new designs as rows in the table above. -->
