# General Concept
A place to plan and track attendance at events and live performances like concerts, festivals, plays, musicals, comedy shows, etc. Allows users with accounts to add to a shared listing of performers (like bands, comedians, etc), venues, show runs (like runs of a musical, music festivals, band tours, etc) to plan ahead for upcoming events and to document past attendance of those shows.

# Features
## Account Features
- Login/Password to view and edit your individual performance listings
- Friends lists allow you to collaborate on events and share schedules
- History reporting will allow you to view filtered lists of events by date range, future/past, type of event, other attendees, venue, and others
- Notes about events, venues, and performances shared as Private, Public, or Friends-only
## General Features
- Allows signed in users to add/edit venues, artists, tours, and shows to a shared data bank
- Performance data can include links to set lists at Setlist.com
- A calendar view allowing users to see events and to toggle on and off Friend accounts in that view to coordinate at festivals and conferences
- Export tool for adding events to a Google Calendar or something similar
# Data Model
Asterisks* below represent required fields
Carets^ below represent connection to another data source
## Venues
Storage for places where events will happen. Every Event must have a Venue associated with it. ==Perhaps this should be "every event must have one or more venues"? Does that needlessly complicate the model for very little value?==
- Unique ID*
- Name*
- City*
- State
- Street Address
- Size (seats/people)
- URL
## Artists
Listing of performers like bands, comedians, etc. Most Events would have one or more Performances that might include an Artist (like a Comedy show would have a Comedian artist, and a Concert would have a Band artist), but might not include an Artist (like a Musical or Play might not list an Artist of any sort--or the theater troupe might be the Artist).
- Unique ID*
- Name*
- Type* (`Band`, `Comedian`, `Performer`, `Troupe`, `Theater Company`, etc)
- URL
## Show Runs
A named series of shows, typically spanning multiple dates, Venues, and perhaps even containing several Artists. Examples include a touring company's shows of a musical or a band's album tour. Not all Performances will be a part of a Show Run, but a Performance would be a part of no more than 1 show run.
- Unique ID*
- Name*
- Artist^
## Performances
A single exhibition of some sort (Concert, Musical, Set, etc) at a single Venue on a single date and time frame. One or more Performances would make up an Event.
- Unique ID*
- Artist^*
- Setlist Link
- Show Run^
- Date *
- Time*
## Events
A collection of Performances across one or more dates
- Unique ID*
- Venue^*
- Performance^*
- Start Date*
- End Date*
- URL
## Notes
Public, Private, or Friends-only notes attached to various types of viewable data
- Unique ID*
- Note Type* (`Event`, `Performance`, `Artist`, `Venue`)
- Privacy* (`Public`, `Friends`, `Private`)
- User ID^*
- Date & Time*
## Users
Store of user accounts and salient user data
- User ID*
- User Name*
- Password*
- Email Address*
- Setlist FM Key
## Relationships
Storing the friendship and follow status for users of the system. For simplicity's sake, there's only two formal relationships between a User and a Relation: following and blocked. This would be depicted as a User's Followers, Following, Blocked, and Friends (where accounts follow each other)
- User ID^*
- Relation^* (User->User ID for the user the relationship describes)
- Relationship* (`following` or `blocked`)
- Change Date*
## Attendance
Stores the past and future event data for a User
- User ID^*
- Performance^*
