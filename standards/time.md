# Time

Craft code uses `chrono` for calendar dates. Do not hand-roll UTC or calendar conversion: days-to-date algorithms and leap-year math. A crate owns only its output format string.

The one exception is range extension. Past the years `chrono` can represent, shift the date by whole 400-year Gregorian cycles (146,097 days, after which the calendar repeats), let `chrono` compute the month and day, and add the cycles back to the year. Splitting seconds into hours, minutes and seconds is plain arithmetic.
