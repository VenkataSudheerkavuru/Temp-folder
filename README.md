IF employee punched after permission start time, THEN
	Calculates the late duration in seconds between punch-in time and shift grace time. 
        if permission start time is after or equal to shift grace time
	1.If employee punched in after permission end time:
Converts permission duration to seconds.          Subtracts permission time from late time
If result is positive, that becomes the new late time.If result is negative, latetime is set to 0

                              

2. else (if employee punched in before permission end time) :
Calculating actual time spent on permission (from permission start-punchin)
Subtracts this time from late duration
Else The shift grace time is after the permission start time andThe shift grace time is not after the permission end time. Which means permission start time before the shift grace start and permission end is after the shift grace start.
(if permission end is also before the shift grace in means we should not even consider this)
If employee punched in after or exactly at permission end time:   Calculates the time difference between permission toTime and shift grace time. Subtracts this duration from the late by
else (if employee punched in before permission end time):
Calculates the time difference between actual punch in time and 
shift grace time. Subtracts this duration from the late by

