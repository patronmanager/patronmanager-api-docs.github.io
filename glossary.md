## PatronManager Glossary


### Ticketable Event (TE)
A Ticketable Event represents a series of performances. For example, "Romeo and Juliet" might run for 10 weeks and have 20 performances. Ticketable Event is often abbreviated "TE". Ticketable Event is the container for one or more Event Instances.
### Event Instance (EI)
An Event Instance represents a single performance of a Ticketable Event. The "Friday, August 3, 2012 7:00 PM EDT" performance of Romeo and Juliet is an example of an Event Instance. Event Instance is often abbreviated "EI". Event Instance is the container for one or more Ticket Allocations
### Ticket Allocation (TA)
Within a single Event Instance, groups of tickets are broken up into Ticket Allocations.  Examples of allocations are: "Orchestra", "Mezzanine", "Balcony". However, Ticket Allocations are not always tied to physical location within the venue. "Holds" and other special access seating are often broken into their own allocation. Inventory is represented on the Ticket Allocation; that is, all tickets within the "Orchestra" allocation may be sold out. Ticket Allocation is often abbreviated "TA". Ticket Allocation is the container for one or more Price Levels.
### Price Level (PL)
Within a Ticket Allocation, tickets can be priced differently. Each of these prices is represented by a Price Level. Examples of Price Levels are "Adult", "Child", "Senior". No inventory is tied to a Price Level. Price Level is often abbreviated "PL".
### Venue 
Venue is an informational field on Event Instance. It represents the physical location that the performance is going to take place and any contact info or special details that the patron should be aware of. Venue is an attribute of Event Instance. This means that each performance of a Ticketable Event can occur at a different location.
### Seating Type
PatronTicket currently supports two seating types: "Pick Your Own Seats" and "General Admission". Seating Type is an attribute of Event Instance. This means that a given Ticketable Event may contain performances of different seating types.
