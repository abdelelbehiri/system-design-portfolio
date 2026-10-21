# System Design Portfolio

A collection of system designs I created for real-world projects, covering architecture decisions, trade-offs, and scalability considerations. Each write-up documents the requirements, architecture, data model, critical design decisions, failure modes, and paths for scaling the system.

## Designs

| Design | Summary | Key challenge |
| --- | --- | --- |
| [Doctor Appointment Booking System](doctor-appointment-booking/) | Search for practitioners and manage appointment reservations, bookings, cancellations, and rescheduling. | Combining eventually consistent search with strongly consistent booking to prevent double reservations. |
| [Retail Stock Broker](stock-broker/) | Stream live stock prices and place, track, and cancel orders routed to an external exchange. | Never duplicating or losing an order when the exchange can time out with an unknown outcome, while reserving buying power before dispatch. |

<!-- Add new designs as rows in the table above. -->
