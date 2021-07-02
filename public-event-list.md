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
	public String smallImagePath; // URL for the small image for this TE
	public String largeImagePath; // URL for the large image for this TE
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
	public String eventName;
	public String detail; // Rich text (HTML) details about this specific EI
	public Dates formattedDates = new Dates();
	public String saleStatus; // "On Sale", "No longer on sale", "Not on sale yet"
	public Boolean earlyAccess = false; // True if early access to the EI is being granted as a Membership benefit
	public String seatingType; // "General Admission" or "Pick Your Own Seats"
	public String contentFormat; // "Standard", "Video on Demand", etc.
	public String purchaseUrl; // URL to the direct purchase page for this EI
	public Boolean soldOut = true; // false unless all TAs are sold out
	public String noSaleMessage; // Message to display if EI is soldOut, 'No longer on sale', or 'Not on sale yet'
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
    "a1A8A000001ZUvKUAW" : {
      "name" : "The Sample Venue Theatre",
      "lastModifiedDate" : "2021-04-02T12:26:16.000Z",
      "id" : "a1A8A000001ZUvKUAW",
      "detail" : "<h3>The Sample Venue Theatre is located at 3639 Williams Lane, Wichita, KS 67202</h3>\r\n<p>This beautiful 360 seat venue features superb acoustic and comfortable seating. For more information, please contact the box office at:</p>\r\n<p>Email: boxoffice@samplevenue.org</p>\r\n<p>Phone:&nbsp;316-665-7204</p>",
      "address" : "3639 Williams Lane\r\nWichita, KS 67202"
    }
  },
  "events" : [ {
    "type" : "Tickets",
    "sortOrder" : 10,
    "smallImagePath" : "https://sillytickets.secure.force.com/ticket/servlet/servlet.ImageServer?id=0155Y000004o9cW&oid=00D5Y000002VmTb&lastMod=1624384870",
    "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/events/a178A000002h2L0QAI",
    "name" : "Romeo & Juliet",
    "largeImagePath" : "https://sillytickets.secure.force.com/ticket/servlet/servlet.ImageServer?id=0155Y000004o9cW&oid=00D5Y000002VmTb&lastMod=1624384870",
    "instances" : [ {
      "venueId" : "a1A8A000001ZUvKUAW",
      "soldOut" : false,
      "seatingType" : "General Admission",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/instances/a0W8A000002uugYUAQ",
      "noSaleMessage" : null,
      "name" : "April 15, 2021, 8 PM",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002uugYUAQ",
      "formattedDates" : {
        "YYYYMMDD" : "20210416",
        "LONG_MONTH_DAY_YEAR" : "April 16, 2021",
        "ISO8601" : "2021-04-16T00:00:00.000Z"
      },
      "eventName" : "Romeo & Juliet",
      "eventId" : "a178A000002h2L0QAI",
      "earlyAccess" : false,
      "detail" : "<h2>Here is an Event Instance with some information in the detail field</h2>\r\n<p>Note this field may contain HTML markup!</p>",
      "custom" : {
        "CustomText__c" : "Here's an EI with a single line of text",
        "CustomMultiselectPicklist__c" : "Emerald;Fuchsia",
        "CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "allocations" : [ {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Orchestra",
        "levels" : [ {
          "price" : 100.00,
          "name" : "Standard",
          "id" : "a168A000002LnOcQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiFQAW"
        }, {
          "price" : 90.00,
          "name" : "Special",
          "id" : "a168A000002LnOfQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiFQAW"
        } ],
        "instanceId" : "a0W8A000002uugYUAQ",
        "id" : "a128A000003JXiFQAW"
      }, {
        "sortOrder" : 2,
        "soldOut" : false,
        "name" : "Mezzanine",
        "levels" : [ {
          "price" : 90.00,
          "name" : "Standard",
          "id" : "a168A000002LnOaQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiGQAW"
        }, {
          "price" : 80.00,
          "name" : "Special",
          "id" : "a168A000002LnOdQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiGQAW"
        } ],
        "instanceId" : "a0W8A000002uugYUAQ",
        "id" : "a128A000003JXiGQAW"
      }, {
        "sortOrder" : 3,
        "soldOut" : false,
        "name" : "Balcony",
        "levels" : [ {
          "price" : 80.00,
          "name" : "Standard",
          "id" : "a168A000002LnObQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiHQAW"
        }, {
          "price" : 70.00,
          "name" : "Special",
          "id" : "a168A000002LnOeQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiHQAW"
        } ],
        "instanceId" : "a0W8A000002uugYUAQ",
        "id" : "a128A000003JXiHQAW"
      } ]
    }, {
      "venueId" : "a1A8A000001ZUvKUAW",
      "soldOut" : false,
      "seatingType" : "General Admission",
      "saleStatus" : "Not on sale yet",
      "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/instances/a0W8A000002uugdUAA",
      "noSaleMessage" : "<p>Hold your horses! Tickets for this performance are not available for sale yet. Coming soon!</p>",
      "name" : "May 15, 2021, 8 PM",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002uugdUAA",
      "formattedDates" : {
        "YYYYMMDD" : "20210516",
        "LONG_MONTH_DAY_YEAR" : "May 16, 2021",
        "ISO8601" : "2021-05-16T00:00:00.000Z"
      },
      "eventName" : "Romeo & Juliet",
      "eventId" : "a178A000002h2L0QAI",
      "earlyAccess" : false,
      "detail" : "<h2>Here is an Event Instance with some information in the detail field</h2>\r\n<p>Note this field may contain HTML markup!</p>",
      "custom" : {
        "CustomText__c" : null,
        "CustomMultiselectPicklist__c" : null,
        "CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "allocations" : [ {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Orchestra",
        "levels" : [ {
          "price" : 100.00,
          "name" : "Standard",
          "id" : "a168A000002LnOmQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiKQAW"
        }, {
          "price" : 90.00,
          "name" : "Special",
          "id" : "a168A000002LnOpQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiKQAW"
        } ],
        "instanceId" : "a0W8A000002uugdUAA",
        "id" : "a128A000003JXiKQAW"
      }, {
        "sortOrder" : 2,
        "soldOut" : false,
        "name" : "Mezzanine",
        "levels" : [ {
          "price" : 90.00,
          "name" : "Standard",
          "id" : "a168A000002LnOoQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiLQAW"
        }, {
          "price" : 80.00,
          "name" : "Special",
          "id" : "a168A000002LnOrQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiLQAW"
        } ],
        "instanceId" : "a0W8A000002uugdUAA",
        "id" : "a128A000003JXiLQAW"
      }, {
        "sortOrder" : 3,
        "soldOut" : false,
        "name" : "Balcony",
        "levels" : [ {
          "price" : 80.00,
          "name" : "Standard",
          "id" : "a168A000002LnOnQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiMQAW"
        }, {
          "price" : 70.00,
          "name" : "Special",
          "id" : "a168A000002LnOqQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiMQAW"
        } ],
        "instanceId" : "a0W8A000002uugdUAA",
        "id" : "a128A000003JXiMQAW"
      } ]
    } ],
    "id" : "a178A000002h2L0QAI",
    "detail" : "<h2>Here is some more detail about this performance</h2>\r\n<p>The detail field can contain HTML markup. Including hyperlinks like this one to&nbsp;<a href=\"https://www.google.com\">Google</a></p>",
    "description" : "<h2>Here's an elaborate description of this performance</h2>\r\n<p>This performance is a \"General Admission\" performance, meaning that seating is on a first come first serve basis. Door's open 60 minutes before the show starts.</p>",
    "custom" : {
      "CustomTextArea__c" : "This is a multi-line custom text area.\r\nIt may contain text with embedded line breaks.\r\nHere another line.",
      "CustomText__c" : "This field may contain a single line of text",
      "CustomCheckbox__c" : true
    },
    "category" : "Play;Drama"
  }, {
    "type" : "Tickets",
    "sortOrder" : 11,
    "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/events/a178A000002h2L5QAI",
    "name" : "Hamlet",
    "instances" : [ {
      "venueId" : "a1A8A000001ZUvKUAW",
      "soldOut" : false,
      "seatingType" : "Pick Your Own Seats",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/instances/a0W8A000002uuk6UAA",
      "noSaleMessage" : null,
      "name" : "May 1, 7 PM",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002uuk6UAA",
      "formattedDates" : {
        "YYYYMMDD" : "20210501",
        "LONG_MONTH_DAY_YEAR" : "May 1, 2021",
        "ISO8601" : "2021-05-01T23:00:00.000Z"
      },
      "eventName" : "Hamlet",
      "eventId" : "a178A000002h2L5QAI",
      "earlyAccess" : false,
      "detail" : "<h2>This is the Event Instance detail field</h2>\r\n<p>It may contain markup</p>",
      "custom" : {
        "CustomText__c" : null,
        "CustomMultiselectPicklist__c" : null,
        "CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "allocations" : [ {
        "sortOrder" : 0,
        "soldOut" : false,
        "name" : "Premium Orchestra",
        "levels" : [ {
          "price" : 100.00,
          "name" : "Retail",
          "id" : "a168A000002LnPQQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JY2TQAW"
        } ],
        "instanceId" : "a0W8A000002uuk6UAA",
        "id" : "a128A000003JY2TQAW"
      }, {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Orchestra",
        "levels" : [ {
          "price" : 95.00,
          "name" : "Retail",
          "id" : "a168A000002LnPVQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JY2UQAW"
        } ],
        "instanceId" : "a0W8A000002uuk6UAA",
        "id" : "a128A000003JY2UQAW"
      }, {
        "sortOrder" : 2,
        "soldOut" : false,
        "name" : "Premium Mezzanine",
        "levels" : [ {
          "price" : 90.00,
          "name" : "Retail",
          "id" : "a168A000002LnPUQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JY2VQAW"
        } ],
        "instanceId" : "a0W8A000002uuk6UAA",
        "id" : "a128A000003JY2VQAW"
      }, {
        "sortOrder" : 3,
        "soldOut" : false,
        "name" : "Mezzanine",
        "levels" : [ {
          "price" : 85.00,
          "name" : "Retail",
          "id" : "a168A000002LnPTQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JY2WQAW"
        } ],
        "instanceId" : "a0W8A000002uuk6UAA",
        "id" : "a128A000003JY2WQAW"
      }, {
        "sortOrder" : 4,
        "soldOut" : false,
        "name" : "Premium Balcony",
        "levels" : [ {
          "price" : 80.00,
          "name" : "Retail",
          "id" : "a168A000002LnPSQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JY2XQAW"
        } ],
        "instanceId" : "a0W8A000002uuk6UAA",
        "id" : "a128A000003JY2XQAW"
      }, {
        "sortOrder" : 5,
        "soldOut" : false,
        "name" : "Balcony",
        "levels" : [ {
          "price" : 75.00,
          "name" : "Retail",
          "id" : "a168A000002LnPRQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JY2YQAW"
        } ],
        "instanceId" : "a0W8A000002uuk6UAA",
        "id" : "a128A000003JY2YQAW"
      } ]
    }, {
      "venueId" : "a1A8A000001ZUvKUAW",
      "soldOut" : false,
      "seatingType" : "Pick Your Own Seats",
      "saleStatus" : "Not on sale yet",
      "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/instances/a0W8A000002uugiUAA",
      "noSaleMessage" : "<p>Here's a custom \"Not On Sale Yet\" message</p>",
      "name" : "June 1, 7 PM",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002uugiUAA",
      "formattedDates" : {
        "YYYYMMDD" : "20210601",
        "LONG_MONTH_DAY_YEAR" : "June 1, 2021",
        "ISO8601" : "2021-06-01T23:00:00.000Z"
      },
      "eventName" : "Hamlet",
      "eventId" : "a178A000002h2L5QAI",
      "earlyAccess" : false,
      "detail" : "<h2>This is the Event Instance detail field</h2>\r\n<p>It may contain markup</p>",
      "custom" : {
        "CustomText__c" : null,
        "CustomMultiselectPicklist__c" : "Orange;Purple",
        "CustomCheckbox__c" : true
      },
      "contentFormat" : "Standard",
      "allocations" : [ {
        "sortOrder" : 0,
        "soldOut" : false,
        "name" : "Premium Orchestra",
        "levels" : [ {
          "price" : 100.00,
          "name" : "Retail",
          "id" : "a168A000002LnPDQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiPQAW"
        } ],
        "instanceId" : "a0W8A000002uugiUAA",
        "id" : "a128A000003JXiPQAW"
      }, {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Orchestra",
        "levels" : [ {
          "price" : 95.00,
          "name" : "Retail",
          "id" : "a168A000002LnPGQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiQQAW"
        } ],
        "instanceId" : "a0W8A000002uugiUAA",
        "id" : "a128A000003JXiQQAW"
      }, {
        "sortOrder" : 2,
        "soldOut" : false,
        "name" : "Premium Mezzanine",
        "levels" : [ {
          "price" : 90.00,
          "name" : "Retail",
          "id" : "a168A000002LnPIQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiRQAW"
        } ],
        "instanceId" : "a0W8A000002uugiUAA",
        "id" : "a128A000003JXiRQAW"
      }, {
        "sortOrder" : 3,
        "soldOut" : false,
        "name" : "Mezzanine",
        "levels" : [ {
          "price" : 85.00,
          "name" : "Retail",
          "id" : "a168A000002LnPKQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiSQAW"
        } ],
        "instanceId" : "a0W8A000002uugiUAA",
        "id" : "a128A000003JXiSQAW"
      }, {
        "sortOrder" : 4,
        "soldOut" : false,
        "name" : "Premium Balcony",
        "levels" : [ {
          "price" : 80.00,
          "name" : "Retail",
          "id" : "a168A000002LnPMQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiTQAW"
        } ],
        "instanceId" : "a0W8A000002uugiUAA",
        "id" : "a128A000003JXiTQAW"
      }, {
        "sortOrder" : 5,
        "soldOut" : false,
        "name" : "Balcony",
        "levels" : [ {
          "price" : 75.00,
          "name" : "Retail",
          "id" : "a168A000002LnPOQA0",
          "fee" : 5.00,
          "allocationId" : "a128A000003JXiUQAW"
        } ],
        "instanceId" : "a0W8A000002uugiUAA",
        "id" : "a128A000003JXiUQAW"
      } ]
    } ],
    "id" : "a178A000002h2L5QAI",
    "detail" : "<h2>Here's an event with HTML detail</h2>\r\n<p>Notice the presence of html tags in this field.</p>",
    "description" : "<h3>Here's an event with a description containing HTML markup</h3>\r\n<p>Notice the html tags in this field.</p>",
    "custom" : {
      "CustomTextArea__c" : null,
      "CustomText__c" : null,
      "CustomCheckbox__c" : true
    },
    "category" : "Concert;Jazz"
  }, {
    "type" : "Subscription",
    "sortOrder" : 15,
    "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/events/a178A000002h2MwQAI",
    "name" : "Test Subscription",
    "instances" : [ {
      "venueId" : null,
      "soldOut" : false,
      "seatingType" : "General Admission",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/instances/a0W8A000002uukVUAQ",
      "noSaleMessage" : null,
      "name" : "4-Show Fixed - Mixed GA / PYOS",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002uukVUAQ",
      "formattedDates" : {
        "YYYYMMDD" : "20210430",
        "LONG_MONTH_DAY_YEAR" : "April 30, 2021",
        "ISO8601" : "2021-04-30T23:00:00.000Z"
      },
      "eventName" : "Test Subscription",
      "eventId" : "a178A000002h2MwQAI",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "CustomText__c" : null,
        "CustomMultiselectPicklist__c" : null,
        "CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "allocations" : [ {
        "sortOrder" : 1,
        "soldOut" : false,
        "name" : "Orchestra",
        "levels" : [ {
          "price" : 310.00,
          "name" : "Subscription",
          "id" : "a168A000002LnPrQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JY5qQAG"
        } ],
        "instanceId" : "a0W8A000002uukVUAQ",
        "id" : "a128A000003JY5qQAG"
      }, {
        "sortOrder" : 2,
        "soldOut" : false,
        "name" : "Mezzanine",
        "levels" : [ {
          "price" : 270.00,
          "name" : "Subscription",
          "id" : "a168A000002LnPwQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JY5vQAG"
        } ],
        "instanceId" : "a0W8A000002uukVUAQ",
        "id" : "a128A000003JY5vQAG"
      }, {
        "sortOrder" : 3,
        "soldOut" : false,
        "name" : "Balcony",
        "levels" : [ {
          "price" : 230.00,
          "name" : "Subscription",
          "id" : "a168A000002LnQ1QAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JY6AQAW"
        } ],
        "instanceId" : "a0W8A000002uukVUAQ",
        "id" : "a128A000003JY6AQAW"
      } ]
    } ],
    "id" : "a178A000002h2MwQAI",
    "detail" : null,
    "description" : null,
    "custom" : {
      "CustomTextArea__c" : null,
      "CustomText__c" : null,
      "CustomCheckbox__c" : false
    },
    "category" : null
  }, {
    "type" : "Membership",
    "sortOrder" : 20,
    "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/instances/a178A000002gtOLQAY",
    "name" : "Membership",
    "instances" : [ {
      "venueId" : null,
      "soldOut" : false,
      "seatingType" : "General Admission",
      "saleStatus" : "On Sale",
      "purchaseUrl" : "https://sillytickets.secure.force.com/ticket/#/instances/a178A000002gtOLQAY",
      "noSaleMessage" : null,
      "name" : "Membership",
      "isPasscodeEligible" : false,
      "id" : "a0W8A000002udGtUAI",
      "formattedDates" : {
        "YYYYMMDD" : "21991231",
        "LONG_MONTH_DAY_YEAR" : "December 31, 2199",
        "ISO8601" : "2199-12-31T08:00:00.000Z"
      },
      "eventName" : "Membership",
      "eventId" : "a178A000002gtOLQAY",
      "earlyAccess" : false,
      "detail" : null,
      "custom" : {
        "CustomText__c" : null,
        "CustomMultiselectPicklist__c" : null,
        "CustomCheckbox__c" : false
      },
      "contentFormat" : "Standard",
      "allocations" : [ {
        "sortOrder" : 0,
        "soldOut" : false,
        "name" : "Membership",
        "levels" : [ {
          "price" : 125.00,
          "name" : "Platinum",
          "id" : "a168A000002LnPmQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JSqAQAW"
        }, {
          "price" : 100.00,
          "name" : "Gold",
          "id" : "a168A000002LnPcQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JSqAQAW"
        }, {
          "price" : 75.00,
          "name" : "Silver",
          "id" : "a168A000002LnPhQAK",
          "fee" : 5.00,
          "allocationId" : "a128A000003JSqAQAW"
        } ],
        "instanceId" : "a0W8A000002udGtUAI",
        "id" : "a128A000003JSqAQAW"
      } ]
    } ],
    "id" : "a178A000002gtOLQAY",
    "detail" : "<h2>Here is the Membership detail value</h2>\r\n<p>This field also may contain HTML markup</p>",
    "description" : "<h2>Here is a Membership with a description</h2>\r\n<p>This may contain HTML markup</p>",
    "custom" : {
      "CustomTextArea__c" : null,
      "CustomText__c" : null,
      "CustomCheckbox__c" : false
    },
    "category" : null
  } ]
}
```
