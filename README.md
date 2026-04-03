# Setlist Collection Manager
Hobby project to collect set lists and links to setlist.fm entries for shows 
that I have gone to and will go to in the future. It's mostly a chance to 
play with AI tools to familiarize myself and to lightweight dust off my 
coding rustiness.

## Features
This is less a feature list and more an aspirational list of things I hope
to add to it as I go:
- Database of Venues, Bands, Shows, Links to setlists on setlist.fm, and Setlists
- Interface to setlist.fm's API
- Ability to provide a date and venue pair to get links to correct shows on 
setlist.fm
- Ability to view the collected list of shows by date, venue, or band
- Ability to pull metrics from attended shows like shows/bands/songs in a 
year, etc
- Ability to edit or add shows/bands/venues/songs manually
- Automatic normalizing of data to source(s); band name, venue name, maybe 
others (not song name)
- Ability to submit a full list of date/venue pairs and have lists of concert 
entries generated automatically
- Ability to export a dump of all songs in order for a show
- Ability to sync attendance at a show on setlist.fm by adding it to the this
database

## Definitions
Some data definitions as I'm using them:
- Venue: The place a show takes place. Data for a venue would include
    - Name
    - City
    - State
- Artist: One of the bands/solo performers performing a set of songs at a show. 
Data for an artist would include
    - Name
    - Notes
- Song: One of the musical performances done during a set by an artist. Data 
for a song would include
    - Name
    - Artist
    - Notes
- Performance: When an artist delivers a series of Songs at a Venue on a Date
- Setlist: A listing of all of the Songs performed by the Artist at one Performance at one Venue on one Date in the order that they were performed
- Show: All of the Performances by all Artists at one Venue on one Date
- Show Setlist: All of the Songs performed by all Artists at a Show in order 
of Performance
