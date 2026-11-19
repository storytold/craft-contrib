# Time

Craft code uses `chrono` for calendar dates. Never hand-roll UTC or calendar conversion: days-to-date algorithms and leap-year math. A crate owns only its output format string.

For dates past the years `chrono` can represent, extend its range: shift the date by whole 400-year Gregorian cycles (146,097 days, after which the calendar repeats), let `chrono` compute the month and day, and add the cycles back to the year. Splitting seconds into hours, minutes and seconds is plain arithmetic.

## craft-launcher

craft-launcher uses `jiff` for its time code, with no `chrono` beside it, until it moves to `chrono` ([storytold/craft-launcher#1526](https://github.com/storytold/craft-launcher/issues/1526)). [RFC-0048](../rfcs/0048-shared-agents-md.md) records this exception. It follows the rest of this standard: no hand-rolled calendar or UTC conversion.
