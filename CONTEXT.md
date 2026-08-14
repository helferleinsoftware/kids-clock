# kids-clock

A fullscreen colour app for a child's bedroom: the screen shows a single colour
that tells the child whether it is still sleeping time or getting-up time. The
colour is driven entirely by the device's own clock.

## Language

**Abschnitt** (Segment):
A named colour that owns a stretch of the day, identified by the time of day it
begins at. Segments tile the full 24 hours as a ring — each one runs until the
next one begins, and the last one of the day wraps around midnight into the
first.
_Avoid_: Zeitpunkt, event, alarm, trigger, schedule entry

**Startzeit** (Start time):
The wall-clock time of day at which a Abschnitt begins. It is a time of day
only — never a date, never an instant.
_Avoid_: Zeitpunkt, timestamp, trigger time

**Aktiver Abschnitt** (Active segment):
The one Abschnitt whose colour the screen is currently showing: the last one
whose Startzeit is at or before the current time, wrapping to the day's final
Abschnitt for times before the first Startzeit.

**Plan**:
The complete ordered ring of Abschnitte. There is exactly one Plan, it applies
to every day alike, and it is the only thing the settings screen edits.
_Avoid_: Schedule, timetable, profile

**Farbfläche** (Colour surface):
The fullscreen view showing the aktiver Abschnitt's colour, with no system
chrome of any kind. The app's entire primary UI.
_Avoid_: Home screen, clock screen
