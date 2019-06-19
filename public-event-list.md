## Event Inventory API

### Overview
First, you should review our [Glossary](glossary.md) so that the terminology used below is clear. 

The Public Event List endpoint provides a machine-readable (JSON) version of the PatronTicket ticketing site used by patrons to purchase tickets. The same logic used to display and hide various entities on the ticketing site is used when generating the payload returned by the endpoint.

### URL
Every PatronManager organization has its own unique domain (ex. https://sillytickets.secure.force.com/ticket). The Public Event List resource is located at 'PatronTicket__PublicApiEventList'. For example: https://sillytickets.secure.force.com/ticket/PatronTicket__PublicApiEventList)

The only valid HTTP method is GET. 

The Public Event List is read-only, requires no authentication and returns the payload as Content-Type: application/json. 

The server will GZIP the response if you provide the proper 'Accepts-Encoding' header on your GET request.

You should be able to load the URL in your web browser and see the JSON payload. If you use a tool like Firefox or Chrome Dev Tools, you should be able to directly browse the JSON entities under the 'Network' tab.

### Document Structure
```javascript
{     
  "venues" : {"Venue id" : {Venue object}},    
  "events" : [ {TicketableEvent object 1 }, {TicketableEvent object 2}]
}
```

The JSON payload is an object containing two keys: "venues" and "events". The values of these keys is represented below.

```java
class TicketableEvent
{
    public Id id;
    public String name;
    public String description; // Rich text (HTML) description of the TE
    public String detail; // Rich text (HTML) details
    public String category; // Arbitrary list of category labels separated by semi-colon
    public Integer sortOrder;
    public String type; // Currently, "Tickets", "Subscription" or "Membership"
    public String purchaseUrl; // URL to the page containing links to all EIs
    public Map<String,Object> custom; //Enumerates fields from the FieldSet defined by settings.TicketableEventPublicFieldSet__c
    public List<EventInstance> instances = new List<EventInstance>();
}

class EventInstance
{
    public Id id;
    public Id eventId;
    public Id venueId;
    public String name;
    public String detail; // Rich text (HTML) details about this specific EI
    public Dates formattedDates = new Dates();
    public String saleStatus; // "On Sale", "No longer on sale", "Not on sale yet"
    public Boolean earlyAccess = false; // True if early access to the EI is being granted as a Membership benefit
    public String seatingType; // "General Admission" or "Pick Your Own Seats"
    public String purchaseUrl; // URL to the direct purchase page for this EI
    public Boolean soldOut = true; // false unless all TAs are sold out
    public String noSaleMessage; // Message to display if EI is soldOut, 'No longer on sale', or 'Not on sale yet'
    public Map<String,Object> custom; //Enumerates fields from the FieldSet defined by settings.EventInstancePublicFieldSet__c
    public List<TicketAllocation> allocations = new List<TicketAllocation>();
}

class Dates
{
    public DateTime ISO8601;
    public String LONG_MONTH_DAY_YEAR; // "September 22, 2012"
    public String YYYYMMDD; // "20120922"
}

class TicketAllocation
{
    public Id id;
    public Id instanceId;
    public String name;
    public Integer sortOrder;
    public List<TicketPriceLevel> levels = new List<TicketPriceLevel>();
    public Boolean soldOut;
}

class TicketPriceLevel
{
    public Id id;
    public Id allocationId;
    public String name;
    public Decimal price;
    public Decimal fee; // Fee that will be added to the price of the ticket
}

class Venue
{
    public Id id;
    public String name;
    public String address;
    public String detail;
    public DateTime lastModifiedDate;
}
```

### Dates
It is expected that the consumer of this API uses the Event Instance "name" when displaying the date and time of the performance. The organization uses the "name" field to convey the performance name in a fashion they see fit (ex. "July 23, 8 PM"). In addition, the Event Instance object contains a formattedDates attribute which is an object containing 3 representations of the instance date. These 3 values should only be used for sorting or grouping the performances by date, never for identifying a specific Event Instance. 

### Sort Order
When displaying the list of TEs, with their EIs, TAs and PLs, the organization expects them to be sorted with the following precedence:
- TicketableEvent.sortOrder
- TicketableEvent.name
- EventInstance.formattedDates.ISO8601
- EventInstance.name
- TicketAllocation.sortOrder
- TicketAllocation.name
- TicketPriceLevel.sortOrder
- TicketPriceLevel.name

Lucky for you, this is the order of the "events" array in the JSON payload so if you iterate through it sequentially, no re-sorting will be necessary.

### On Sale vs. Sold Out vs. No Longer On Sale vs. Not On Sale Yet
Every PatronManager organization has the option of deciding whether inventory that's 'No longer on sale', 'Sold Out', or 'Not on sale yet' should continue to be displayed. By default, 'Sold Out', 'No longer on sale', and 'Not on sale yet' EIs will disappear from the response payload. If the organization sets custom messages to be displayed in these three states, it will be included in the 'noSaleMessage' on EventInstance. 

There is a 'soldOut' attribute on both TA and EI. 'soldOut' on EI will always be false unless all TAs are sold out, in which case the EI will have 'soldOut' set to true and the 'allocations' collection will be empty.

Suggested pseudocode looks like this:
```javascript
if (ei.saleStatus == 'On Sale' && !ei.soldOut) {
    print "<a href='" + ei.purchaseUrl + "'>Click here to buy tickets for " + ei.name + "</a>";
} else  { 
    print ei.name + " - " + ei.noSaleMessage;
}
```

### Custom Fields
You can add custom fields to both Ticketable Event and Event Instance that you wish to have included in the Public Event List payload. The fields must first be added to the respective SObject (PatronTicket__TicketableEvent__c or PatronTicket__EventInstance__c) via the Setup -> Create -> Objects UI.

Any custom fields added to Ticketable Event or Event Instance immediately become visible on the PatronManager Event Inventory UI. There will be a new section called "Custom Fields" visible at the bottom of the Ticketable Event and Event Instance Edit and Detail pages.

To get the custom included in the Public Event List payload, first create a Field Set on Ticketable Event or Event Instance and add the fields you wish to expose to that Field Set. Next, please contact PatronManager Client Services -- they will enable the Field Set for public view so that any fields enumerated in the Field Set are included in the "custom" key for both the TicketableEvent and EventInstance JSON schema.


### Example
```javascript
{
  "venues": {
    "a05800000070JMaAAM": {
      "id": "a05800000070JMaAAM",
      "name": "Falcon Theatre",
      "lastModifiedDate": "2012-08-06T16:28:39.000Z",
      "detail": "<p>4252 Riverside Dr. <br />Burbank, CA 91505</p>\n<p>(818) 955-8101 <br /><br />Corner of Rose St. and Riverside Dr., across from the historic Bob's Big Boy restaurant in Toluca Lake. <br /><br />Free parking available.<br /><br />All sales final, no refunds, no exchanges.</p>",
      "address": null
    },
    "a0580000007STVmAAO": {
      "id": "a0580000007STVmAAO",
      "name": "Wide Theatre",
      "lastModifiedDate": "2012-08-03T18:10:47.000Z",
      "detail": "<h2>A Wide Theatre for Testing Purposes</h2>",
      "address": "7 Research Drive\r\nWoodbridge CT, 06525"
    }
  },
  "events": [
    {
      "id": "a018000000N2wwSAAR",
      "name": "The Tom Show",
      "type": "Tickets",
      "sortOrder": 0,
      "purchaseUrl": "https://sillytickets.secure.force.com/ticket#details_a018000000N2wwSAAR",
      "description": "<p>He's the coolest guy around!</p>",
      "detail": "<p>It's the greatest show on Earth!</p>",
      "category": "Play;Comedy",
      "instances": [
        {
          "id": "a068000000OgviCAAR",
          "name": "Wide Theatre, Sept. 30, 2012",
          "eventId": "a018000000N2wwSAAR",
          "venueId": "a0580000007STVmAAO",
          "soldOut": false,
          "seatingType": "Pick Your Own Seats",
          "saleStatus": "On Sale",
          "purchaseUrl": "https://sillytickets.secure.force.com/ticket#sections_a068000000OgviCAAR",
          "noSaleMessage": null,
          "formattedDates": {
            "YYYYMMDD": "20121001",
            "LONG_MONTH_DAY_YEAR": "October 1, 2012",
            "ISO8601": "2012-10-01T00:00:00.000Z"
          },
          "detail": null,
          "allocations": [
            {
              "id": "a078000000GfruEAAR",
              "name": "Floor",
              "instanceId": "a068000000OgviCAAR",
              "sortOrder": 0,
              "soldOut": false,
              "levels": [
                {
                  "id": "a048000000HdzTFAAZ",
                  "name": "Adult",
                  "allocationId": "a078000000GfruEAAR",
                  "sortOrder": null,
                  "price": 40,
                  "fee": 4
                },
                {
                  "id": "a048000000HdzTKAAZ",
                  "name": "Child/Senior",
                  "allocationId": "a078000000GfruEAAR",
                  "sortOrder": null,
                  "price": 30,
                  "fee": 4
                }
              ]
            },
            {
              "id": "a078000000GfsKCAAZ",
              "name": "Friends of Tom",
              "instanceId": "a068000000OgviCAAR",
              "sortOrder": 1,
              "soldOut": false,
              "levels": [
                {
                  "id": "a048000000Hdzf2AAB",
                  "name": "Friends of Tom",
                  "allocationId": "a078000000GfsKCAAZ",
                  "sortOrder": null,
                  "price": 10,
                  "fee": 2
                }
              ]
            }
          ]
        },
        {
          "id": "a068000000OjDHlAAN",
          "name": "Falcon Theatre, Oct. 5, 2012",
          "eventId": "a018000000N2wwSAAR",
          "venueId": "a05800000070JMaAAM",
          "soldOut": false,
          "seatingType": "Pick Your Own Seats",
          "saleStatus": "On Sale",
          "purchaseUrl": "https://sillytickets.secure.force.com/ticket#sections_a068000000OjDHlAAN",
          "noSaleMessage": null,
          "formattedDates": {
            "YYYYMMDD": "20121006",
            "LONG_MONTH_DAY_YEAR": "October 6, 2012",
            "ISO8601": "2012-10-06T00:00:00.000Z"
          },
          "detail": null,
          "allocations": [
            {
              "id": "a078000000Gh8mCAAR",
              "name": "Side",
              "instanceId": "a068000000OjDHlAAN",
              "sortOrder": 0,
              "soldOut": false,
              "levels": [
                {
                  "id": "a048000000IuI3kAAF",
                  "name": "Fri/Sat/Sun",
                  "allocationId": "a078000000Gh8mCAAR",
                  "sortOrder": null,
                  "price": 39.5,
                  "fee": 4
                }
              ]
            },
            {
              "id": "a078000000Gh8mDAAR",
              "name": "Center",
              "instanceId": "a068000000OjDHlAAN",
              "sortOrder": 1,
              "soldOut": false,
              "levels": [
                {
                  "id": "a048000000IuI3lAAF",
                  "name": "Fri/Sat/Sun",
                  "allocationId": "a078000000Gh8mDAAR",
                  "sortOrder": null,
                  "price": 42,
                  "fee": 4
                }
              ]
            },
            {
              "id": "a078000000Gh8mFAAR",
              "name": "Accessible Side",
              "instanceId": "a068000000OjDHlAAN",
              "sortOrder": 3,
              "soldOut": false,
              "levels": [
                {
                  "id": "a048000000IuI3nAAF",
                  "name": "Fri/Sat/Sun",
                  "allocationId": "a078000000Gh8mFAAR",
                  "sortOrder": null,
                  "price": 39.5,
                  "fee": 4
                }
              ]
            },
            {
              "id": "a078000000Gh8oIAAR",
              "name": "Accessible",
              "instanceId": "a068000000OjDHlAAN",
              "sortOrder": 4,
              "soldOut": false,
              "levels": [
                {
                  "id": "a048000000IuI3oAAF",
                  "name": "Fri/Sat/Sun",
                  "allocationId": "a078000000Gh8oIAAR",
                  "sortOrder": null,
                  "price": 42,
                  "fee": 4
                }
              ]
            }
          ]
        }
      ]
    },
    {
      "id": "a018000000RjkZ1AAJ",
      "name": "Twelve Angry Men - 2012",
      "type": "Tickets",
      "sortOrder": 41,
      "purchaseUrl": "https://sillytickets.secure.force.com/ticket#details_a018000000RjkZ1AAJ",
      "description": null,
      "detail": null,
      "category": "Play;Drama",
      "instances": [
        {
          "id": "a068000000X6rHzAAJ",
          "name": "November 2, 2012",
          "eventId": "a018000000RjkZ1AAJ",
          "venueId": null,
          "soldOut": false,
          "seatingType": "General Admission",
          "saleStatus": "No longer on sale",
          "purchaseUrl": "https://sillytickets.secure.force.com/ticket#sections_a068000000X6rHzAAJ",
          "noSaleMessage": "The sale date has passed for this instance",
          "formattedDates": {
            "YYYYMMDD": "20121102",
            "LONG_MONTH_DAY_YEAR": "November 2, 2012",
            "ISO8601": "2012-11-02T23:00:00.000Z"
          },
          "detail": null,
          "allocations": []
        },
        {
          "id": "a068000000X6rIOAAZ",
          "name": "November 9, 2012",
          "eventId": "a018000000RjkZ1AAJ",
          "venueId": null,
          "soldOut": false,
          "seatingType": "General Admission",
          "saleStatus": "On Sale",
          "purchaseUrl": "https://sillytickets.secure.force.com/ticket#sections_a068000000X6rIOAAZ",
          "noSaleMessage": null,
          "formattedDates": {
            "YYYYMMDD": "20121110",
            "LONG_MONTH_DAY_YEAR": "November 10, 2012",
            "ISO8601": "2012-11-10T00:00:00.000Z"
          },
          "detail": null,
          "allocations": [
            {
              "id": "a078000000JmYgtAAF",
              "name": "General Admission",
              "instanceId": "a068000000X6rIOAAZ",
              "sortOrder": 1,
              "soldOut": false,
              "levels": [
                {
                  "id": "a048000000KEqUTAA1",
                  "name": "Standard",
                  "allocationId": "a078000000JmYgtAAF",
                  "sortOrder": null,
                  "price": 30,
                  "fee": 4
                }
              ]
            },
            {
              "id": "a078000000JmYgsAAF",
              "name": "Wheelchair",
              "instanceId": "a068000000X6rIOAAZ",
              "sortOrder": 2,
              "soldOut": false,
              "levels": [
                {
                  "id": "a048000000KEqUUAA1",
                  "name": "Standard",
                  "allocationId": "a078000000JmYgsAAF",
                  "sortOrder": null,
                  "price": 30,
                  "fee": 4
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```


