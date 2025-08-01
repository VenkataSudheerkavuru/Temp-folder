1. Method: calculateFixedWH

Purpose

To calculate the total work hours for a given day using the employee’s first punch and last punch, while considering permissions.

Pseudocode

FUNCTION calculateFixedWH(workHourPolicy, currentDayShift, currentDate,
                          workHourDetails, firstPunch, lastPunch, shiftProcessingContextModel):

    totalFWH = getFixedWorkHours(firstPunch, lastPunch, currentDate, workHourPolicy)

    IF totalFWH > 0:
        workHourDetails.totalWorkHours = totalFWH
        workHourDetails.totalWorkHoursFormatted = convertSecondsToHHMMSS(totalFWH)
    ELSE:
        workHourDetails.totalWorkHours = 0
        workHourDetails.totalWorkHoursFormatted = "00:00:00"

    RETURN workHourDetails

Explanation

Delegates actual duration calculation to getFixedWorkHours.

Stores final work hours (raw seconds + formatted).

Ensures no negative totals.



---

2. Method: getFixedWorkHours

Purpose

Computes base work hours from first and last punch, and adjusts it by adding approved permission hours.

Formula

Base Duration = (Last Punch Time − First Punch Time)
Total Work Hours = Base Duration + Credited Permission Hours

Pseudocode

FUNCTION getFixedWorkHours(firstPunch, lastPunch, currentDate, workHourPolicy):

    inTime  = firstPunch.time
    outTime = lastPunch.time

    baseDuration = (outTime - inTime) in seconds

    totalFWH = baseDuration

    totalFWH = getFixedPermissionHours(currentDate, totalFWH, firstPunch, lastPunch)

    RETURN totalFWH

Explanation

Calculates raw duration between first and last punch.

Calls getFixedPermissionHours to add permission-based credits.



---

3. Method: getFixedPermissionHours

Purpose

To credit approved permission hours into total work hours, so that excused time counts as attendance.

Key Policy Flag

permissionDetails.isAddHoursToTheAttendance

Only permissions with this flag set to true are credited.

HR policy controls which permission types add hours.


Pseudocode

FUNCTION getFixedPermissionHours(currentDate, totalFWR, firstPunch, lastPunch):

    permissions = searchApprovedPermissions(employeeId, currentDate)

    FOR each permission IN permissions:

        IF permission.isAddHoursToTheAttendance:

            permFrom = permission.start
            permTo   = permission.end

            // Case 1: First Punch inside permission
            IF firstPunch > permFrom AND firstPunch < permTo:
                credit = firstPunch - permFrom
                totalFWR = totalFWR + credit

            // Case 2: First Punch after permission ended
            ELSE IF firstPunch > permFrom:
                credit = permission.differenceHours
                totalFWR = totalFWR + credit

            // Case 3: Last Punch inside permission
            IF lastPunch > permFrom AND lastPunch < permTo:
                credit = permTo - lastPunch
                totalFWR = totalFWR + credit

            // Case 4: Last Punch before permission starts
            ELSE IF lastPunch <= permFrom:
                credit = permission.differenceHours
                totalFWR = totalFWR + credit

    RETURN totalFWR


---
