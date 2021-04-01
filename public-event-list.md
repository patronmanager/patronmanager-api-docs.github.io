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
    public Boolean soldOut = true; // False unless all TAs are sold out
    public String noSaleMessage; // Message to display if EI is soldOut, 'No longer on sale', or 'Not on sale yet'
    public String contentFormat; // "Standard" (a live in-person EI), "Livestream (RTMP)", "Livestream (External)", or "Video on Demand"
    public Boolean isPasscodeEligible = false; // True if the Event Instance is available for purchase prior to its public on-sale with the use of a Passcode
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
  "venues" : {
    "a1A8A000001ZPFUUA4" : {
      "name" : "Allentown Symphony Hall",
      "lastModifiedDate" : "2021-03-26T16:55:08.000Z",
      "id" : "a1A8A000001ZPFUUA4",
      "detail" : null,
      "address" : null
    },
    "a1A8A000001ZPAUUA4" : {
      "name" : "The Barn",
      "lastModifiedDate" : "2021-03-26T16:20:20.000Z",
      "id" : "a1A8A000001ZPAUUA4",
      "detail" : null,
      "address" : null
    }
  },
  "events" : [ {
    "type" : "Tickets",
    "sortOrder" : 1,
    "smallImagePath" : null,
    "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/events/a178A000002gtOIQAY",
    "name" : "Test Event",
    "largeImagePath" : null,
    "instances" : [ {
      "venueId" : "a1A8A000001ZPAUUA4",
      "soldOut" : false,
      "seatingType" : "Pick Your Own Seats",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002udGOUAY",
      "noSaleMessage" : null,
      "name" : "Access Code Event (PYOS) Dec. 25th 2029 8pm",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002udGOUAY",
      "formattedDates" : {
        "YYYYMMDD" : "20291225",
        "LONG_MONTH_DAY_YEAR" : "December 25, 2029",
        "ISO8601" : "2029-12-25T14:09:00.000Z"
      },
      "eventName" : "Test Event",
      "eventId" : "a178A000002gtOIQAY",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "appliedPasscode" : null,
      "allocations" : [ {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Tier 2",
        "levels" : [ ],
        "instanceId" : "a0W8A000002udGOUAY",
        "id" : "a128A000003JSqgQAG"
      } ]
    }, {
      "venueId" : "a1A8A000001ZPAUUA4",
      "soldOut" : false,
      "seatingType" : "Pick Your Own Seats",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002udGpUAI",
      "noSaleMessage" : null,
      "name" : "Test Event (PYOC) Dec. 25th 2029 8pm",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002udGpUAI",
      "formattedDates" : {
        "YYYYMMDD" : "20291226",
        "LONG_MONTH_DAY_YEAR" : "December 26, 2029",
        "ISO8601" : "2029-12-26T04:00:00.000Z"
      },
      "eventName" : "Test Event",
      "eventId" : "a178A000002gtOIQAY",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "appliedPasscode" : null,
      "allocations" : [ {
        "sortOrder" : 0,
        "soldOut" : false,
        "name" : "Tier 1",
        "levels" : [ ],
        "instanceId" : "a0W8A000002udGpUAI",
        "id" : "a128A000003JSqFQAW"
      }, {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Tier 2",
        "levels" : [ ],
        "instanceId" : "a0W8A000002udGpUAI",
        "id" : "a128A000003JSqTQAW"
      } ]
    }, {
      "venueId" : null,
      "soldOut" : false,
      "seatingType" : "General Admission",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002udGUUAY",
      "noSaleMessage" : null,
      "name" : "NOSY Event (GA) Dec. 26th 2029 8pm",
      "isPasscodeEligible" : true,
      "id" : "a0W8A000002udGUUAY",
      "formattedDates" : {
        "YYYYMMDD" : "20291227",
        "LONG_MONTH_DAY_YEAR" : "December 27, 2029",
        "ISO8601" : "2029-12-27T04:00:00.000Z"
      },
      "eventName" : "Test Event",
      "eventId" : "a178A000002gtOIQAY",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "appliedPasscode" : null,
      "allocations" : [ ]
    } ],
    "id" : "a178A000002gtOIQAY",
    "detail" : null,
    "description" : null,
    "custom" : {
      "PatronTicket__XYZZY_CustomTextArea__c" : null,
      "PatronTicket__XYZZY_CustomText__c" : null,
      "PatronTicket__XYZZY_CustomCheckbox__c" : false
    },
    "category" : "Play;Comedy"
  }, {
    "type" : "Tickets",
    "sortOrder" : 2,
    "smallImagePath" : null,
    "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/events/a178A000002gtaAQAQ",
    "name" : "The Tom Show - 2021",
    "largeImagePath" : null,
    "instances" : [ {
      "venueId" : null,
      "soldOut" : true,
      "seatingType" : "General Admission",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002upGDUAY",
      "noSaleMessage" : "<p>Custom \"Sold Out\" message</p>",
      "name" : "Sold Out",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002upGDUAY",
      "formattedDates" : {
        "YYYYMMDD" : "20210801",
        "LONG_MONTH_DAY_YEAR" : "August 1, 2021",
        "ISO8601" : "2021-08-01T13:10:00.000Z"
      },
      "eventName" : "The Tom Show - 2021",
      "eventId" : "a178A000002gtaAQAQ",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "appliedPasscode" : null,
      "allocations" : [ ]
    }, {
      "venueId" : null,
      "soldOut" : false,
      "seatingType" : "General Admission",
      "saleStatus" : "No longer on sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002upG8UAI",
      "noSaleMessage" : "<p>Custom \"No Longer On Sale\" message</p>",
      "name" : "No Longer On Sale",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002upG8UAI",
      "formattedDates" : {
        "YYYYMMDD" : "20210801",
        "LONG_MONTH_DAY_YEAR" : "August 1, 2021",
        "ISO8601" : "2021-08-01T13:10:00.000Z"
      },
      "eventName" : "The Tom Show - 2021",
      "eventId" : "a178A000002gtaAQAQ",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "appliedPasscode" : null,
      "allocations" : [ {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Public",
        "levels" : [ {
          "price" : 50.00,
          "name" : "Public",
          "id" : "a168A000002Ln9wQAC",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXKTQA4"
        } ],
        "instanceId" : "a0W8A000002upG8UAI",
        "id" : "a128A000003JXKTQA4"
      } ]
    }, {
      "venueId" : null,
      "soldOut" : false,
      "seatingType" : "General Admission",
      "saleStatus" : "Not on sale yet",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002upG3UAI",
      "noSaleMessage" : "<p>Custom \"Not On Sale Yet\" message</p>",
      "name" : "Not On Sale Yet",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002upG3UAI",
      "formattedDates" : {
        "YYYYMMDD" : "20210801",
        "LONG_MONTH_DAY_YEAR" : "August 1, 2021",
        "ISO8601" : "2021-08-01T13:10:00.000Z"
      },
      "eventName" : "The Tom Show - 2021",
      "eventId" : "a178A000002gtaAQAQ",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "appliedPasscode" : null,
      "allocations" : [ {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Public",
        "levels" : [ {
          "price" : 50.00,
          "name" : "Public",
          "id" : "a168A000002Ln9rQAC",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXKOQA4"
        } ],
        "instanceId" : "a0W8A000002upG3UAI",
        "id" : "a128A000003JXKOQA4"
      } ]
    }, {
      "venueId" : "a1A8A000001ZPFUUA4",
      "soldOut" : false,
      "seatingType" : "Pick Your Own Seats",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002udJFUAY",
      "noSaleMessage" : null,
      "name" : "Allentown Symphony Hall, December 1, 8 PM",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002udJFUAY",
      "formattedDates" : {
        "YYYYMMDD" : "20211202",
        "LONG_MONTH_DAY_YEAR" : "December 2, 2021",
        "ISO8601" : "2021-12-02T01:00:00.000Z"
      },
      "eventName" : "The Tom Show - 2021",
      "eventId" : "a178A000002gtaAQAQ",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "appliedPasscode" : null,
      "allocations" : [ {
        "sortOrder" : 0,
        "soldOut" : false,
        "name" : "Price Level A",
        "levels" : [ {
          "price" : 100.00,
          "name" : "Standard",
          "id" : "a168A000002LdfAQAS",
          "fee" : 5.00,
          "allocationId" : "a128A000003JSs9QAG"
        } ],
        "instanceId" : "a0W8A000002udJFUAY",
        "id" : "a128A000003JSs9QAG"
      }, {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Price Level B",
        "levels" : [ {
          "price" : 90.00,
          "name" : "Standard",
          "id" : "a168A000002LdfFQAS",
          "fee" : 5.00,
          "allocationId" : "a128A000003JSsAQAW"
        } ],
        "instanceId" : "a0W8A000002udJFUAY",
        "id" : "a128A000003JSsAQAW"
      }, {
        "sortOrder" : 2,
        "soldOut" : false,
        "name" : "Price Level C",
        "levels" : [ {
          "price" : 80.00,
          "name" : "Standard",
          "id" : "a168A000002LdfKQAS",
          "fee" : 5.00,
          "allocationId" : "a128A000003JSsBQAW"
        } ],
        "instanceId" : "a0W8A000002udJFUAY",
        "id" : "a128A000003JSsBQAW"
      }, {
        "sortOrder" : 3,
        "soldOut" : false,
        "name" : "Price Level D",
        "levels" : [ {
          "price" : 70.00,
          "name" : "Standard",
          "id" : "a168A000002LdfMQAS",
          "fee" : 5.00,
          "allocationId" : "a128A000003JSsCQAW"
        } ],
        "instanceId" : "a0W8A000002udJFUAY",
        "id" : "a128A000003JSsCQAW"
      }, {
        "sortOrder" : 4,
        "soldOut" : false,
        "name" : "Orchestra Boxes",
        "levels" : [ {
          "price" : 95.00,
          "name" : "Standard",
          "id" : "a168A000002LdfPQAS",
          "fee" : 5.00,
          "allocationId" : "a128A000003JSsDQAW"
        } ],
        "instanceId" : "a0W8A000002udJFUAY",
        "id" : "a128A000003JSsDQAW"
      }, {
        "sortOrder" : 5,
        "soldOut" : false,
        "name" : "Mezzanine Boxes",
        "levels" : [ {
          "price" : 75.00,
          "name" : "Standard",
          "id" : "a168A000002LdfQQAS",
          "fee" : 5.00,
          "allocationId" : "a128A000003JSsEQAW"
        } ],
        "instanceId" : "a0W8A000002udJFUAY",
        "id" : "a128A000003JSsEQAW"
      } ]
    } ],
    "id" : "a178A000002gtaAQAQ",
    "detail" : null,
    "description" : null,
    "custom" : {
      "PatronTicket__XYZZY_CustomTextArea__c" : null,
      "PatronTicket__XYZZY_CustomText__c" : null,
      "PatronTicket__XYZZY_CustomCheckbox__c" : false
    },
    "category" : null
  }, {
    "type" : "Tickets",
    "sortOrder" : 3,
    "smallImagePath" : null,
    "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/events/a178A000002gtOQQAY",
    "name" : "Virtual Events",
    "largeImagePath" : null,
    "instances" : [ {
      "venueId" : null,
      "soldOut" : false,
      "seatingType" : "General Admission",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002udIlUAI",
      "noSaleMessage" : null,
      "name" : "VOD - How to Tie the Nail Knot",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002udIlUAI",
      "formattedDates" : {
        "YYYYMMDD" : "20220326",
        "LONG_MONTH_DAY_YEAR" : "March 26, 2022",
        "ISO8601" : "2022-03-26T16:45:00.000Z"
      },
      "eventName" : "Virtual Events",
      "eventId" : "a178A000002gtOQQAY",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Video on Demand",
      "appliedPasscode" : null,
      "allocations" : [ {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "VOD",
        "levels" : [ {
          "price" : 5.00,
          "name" : "Standard",
          "id" : "a168A000002Ldf0QAC",
          "fee" : 0.00,
          "allocationId" : "a128A000003JSrzQAG"
        } ],
        "instanceId" : "a0W8A000002udIlUAI",
        "id" : "a128A000003JSrzQAG"
      } ]
    }, {
      "venueId" : null,
      "soldOut" : false,
      "seatingType" : "General Admission",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002udJAUAY",
      "noSaleMessage" : null,
      "name" : "VOD - The Only Fishing Knot You Need",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002udJAUAY",
      "formattedDates" : {
        "YYYYMMDD" : "20220326",
        "LONG_MONTH_DAY_YEAR" : "March 26, 2022",
        "ISO8601" : "2022-03-26T16:48:00.000Z"
      },
      "eventName" : "Virtual Events",
      "eventId" : "a178A000002gtOQQAY",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Video on Demand",
      "appliedPasscode" : null,
      "allocations" : [ {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "VOD",
        "levels" : [ {
          "price" : 5.00,
          "name" : "Standard",
          "id" : "a168A000002Ldf5QAC",
          "fee" : 0.00,
          "allocationId" : "a128A000003JSs4QAG"
        } ],
        "instanceId" : "a0W8A000002udJAUAY",
        "id" : "a128A000003JSs4QAG"
      } ]
    } ],
    "id" : "a178A000002gtOQQAY",
    "detail" : null,
    "description" : null,
    "custom" : {
      "PatronTicket__XYZZY_CustomTextArea__c" : null,
      "PatronTicket__XYZZY_CustomText__c" : null,
      "PatronTicket__XYZZY_CustomCheckbox__c" : false
    },
    "category" : null
  }, {
    "type" : "Tickets",
    "sortOrder" : 4,
    "smallImagePath" : null,
    "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/events/a178A000002gtOKQAY",
    "name" : "Test Event - 2021",
    "largeImagePath" : null,
    "instances" : [ {
      "venueId" : "a1A8A000001ZPAUUA4",
      "soldOut" : false,
      "seatingType" : "Pick Your Own Seats",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002udGeUAI",
      "noSaleMessage" : null,
      "name" : "PYOS Fulfillment, December 28, 8 PM",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002udGeUAI",
      "formattedDates" : {
        "YYYYMMDD" : "20211229",
        "LONG_MONTH_DAY_YEAR" : "December 29, 2021",
        "ISO8601" : "2021-12-29T04:00:00.000Z"
      },
      "eventName" : "Test Event - 2021",
      "eventId" : "a178A000002gtOKQAY",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "appliedPasscode" : null,
      "allocations" : [ {
        "sortOrder" : 0,
        "soldOut" : false,
        "name" : "Tier 1",
        "levels" : [ ],
        "instanceId" : "a0W8A000002udGeUAI",
        "id" : "a128A000003JSqDQAW"
      }, {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Tier 2",
        "levels" : [ ],
        "instanceId" : "a0W8A000002udGeUAI",
        "id" : "a128A000003JSqaQAG"
      } ]
    }, {
      "venueId" : "a1A8A000001ZPAUUA4",
      "soldOut" : false,
      "seatingType" : "Pick Your Own Seats",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sandbox-littledipper-aries-5925-1786f2f1715.cs45.force.com/ticket/#/instances/a0W8A000002udGhUAI",
      "noSaleMessage" : null,
      "name" : "PYOS Fulfillment, December 29, 8 PM",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002udGhUAI",
      "formattedDates" : {
        "YYYYMMDD" : "20211230",
        "LONG_MONTH_DAY_YEAR" : "December 30, 2021",
        "ISO8601" : "2021-12-30T04:00:00.000Z"
      },
      "eventName" : "Test Event - 2021",
      "eventId" : "a178A000002gtOKQAY",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "PatronTicket__XYZZY_CustomText__c" : null,
        "PatronTicket__XYZZY_CustomMultiselectPicklist__c" : null,
        "PatronTicket__XYZZY_CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "appliedPasscode" : null,
      "allocations" : [ {
        "sortOrder" : 0,
        "soldOut" : false,
        "name" : "Tier 1",
        "levels" : [ ],
        "instanceId" : "a0W8A000002udGhUAI",
        "id" : "a128A000003JSqNQAW"
      }, {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Tier 2",
        "levels" : [ ],
        "instanceId" : "a0W8A000002udGhUAI",
        "id" : "a128A000003JSqZQAW"
      } ]
    } ],
    "id" : "a178A000002gtOKQAY",
    "detail" : null,
    "description" : null,
    "custom" : {
      "PatronTicket__XYZZY_CustomTextArea__c" : null,
      "PatronTicket__XYZZY_CustomText__c" : null,
      "PatronTicket__XYZZY_CustomCheckbox__c" : false
    },
    "category" : "Play;Comedy"
  } ]
}
```
