private WorkHourDetails calculateEarlyBy(ShiftDetails currentDayShift,
      WorkHourDetails workHourDetails, PunchRequest lastPunch, LocalDateTime currentDate,PermissionDetailsList permissionDetailsList,WorkHourPolicyDetails workHourPolicy,ShiftProcessingContextModel shiftProcessingContextModel) {
    GraceTime graceTime = currentDayShift.getGraceTime();
    ShiftTiming shiftTiming = currentDayShift.getShiftTiming();
    LocalDateTime outPunchDateTime = lastPunch.getLocalPunchTime();
    LocalTime shiftEnd = LocalTime
        .parse(shiftTiming.getEndTime(), DateTimeFormatter.ofPattern("H:mm"));
    LocalTime shiftStart = LocalTime
        .parse(shiftTiming.getStartTime(), DateTimeFormatter.ofPattern("H:mm"));
    LocalTime shiftOutGraceTime = calculateShiftOutGrace(graceTime, shiftEnd);
    //In case of night shifts(shift ends at 8am, employee leaves early)
    //if (shiftEnd.isBefore(shiftStart)) {
    if (shiftEnd.isBefore(shiftStart) && (shiftOutGraceTime.isBefore(shiftEnd) || shiftOutGraceTime.equals(shiftEnd))) {
      currentDate = currentDate.plusDays(1);
    }
    LocalDateTime shiftOutGrace = currentDate.toLocalDate().atTime(shiftOutGraceTime);
    long earlyBySeconds = 0;

    if (outPunchDateTime.compareTo(shiftOutGrace) < 0) {
      earlyBySeconds = Math.abs(Duration
          .between(currentDate.toLocalDate().atTime(shiftEnd), lastPunch.getLocalPunchTime())
          .toMillis()) / 1000;

    if (permissionDetailsList.getTotalCount() != null && permissionDetailsList.getTotalCount() > 0
        && earlyBySeconds > 0) {
      try {
        for (int i = 0; i < permissionDetailsList.getTotalCount(); i++) {
          PermissionDetails permissionDetails = permissionDetailsList.getPermissionDetails().get(i);
          LocalTime permissionFromTime = LocalTime
              .parse(permissionDetails.getFromTime(), DateTimeFormatter.ofPattern("H:mm"));
          LocalDateTime permissionFromDateTime = currentDate.toLocalDate().atTime(permissionFromTime);
          LocalTime permissionToTime = LocalTime
              .parse(permissionDetails.getToTime(), DateTimeFormatter.ofPattern("H:mm"));
          LocalDateTime permissionToDateTime = currentDate.toLocalDate().atTime(permissionToTime);
          if (outPunchDateTime.compareTo(permissionFromDateTime) <= 0) {
            earlyBySeconds = Math.abs(Duration
                    .between(currentDate.toLocalDate().atTime(shiftOutGraceTime), lastPunch.getLocalPunchTime())
                    .toMillis()) / 1000;
            if (shiftOutGrace.compareTo(permissionToDateTime) >= 0) {
              LocalTime permissionTime = LocalTime.parse(permissionDetails.getDifferenceHours(),
                  DateTimeFormatter.ofPattern("H:mm:ss"));
              String permisionTimeInSeconds = permissionTime.toString();
              long totalPermissionSeconds = earlyBySeconds - DateUtil.convertHhMmSsToSeconds(permisionTimeInSeconds);
              if (totalPermissionSeconds >= 0) {
                earlyBySeconds = totalPermissionSeconds;
              } else {
                earlyBySeconds = 0;
              }
            } else if (shiftOutGrace.isAfter(permissionFromDateTime)
                && !shiftOutGrace.isAfter(permissionToDateTime)) {
              long totalPT = DateUtil.convertHhMmSsToSeconds(
                  shiftOutGrace.format(DateTimeFormatter.ofPattern("H:mm:ss"))) - DateUtil
                  .convertHhMmSsToSeconds(permissionFromTime + ":00");
              earlyBySeconds = earlyBySeconds - totalPT;
            }
          } else if (outPunchDateTime.isAfter(permissionFromDateTime)
              && outPunchDateTime.isBefore(permissionToDateTime)) {
            earlyBySeconds = Math.abs(Duration
                    .between(currentDate.toLocalDate().atTime(shiftOutGraceTime), lastPunch.getLocalPunchTime())
                    .toMillis()) / 1000;
            if (!shiftOutGrace.isBefore(permissionToDateTime)) {
              long totalPT = DateUtil.convertHhMmSsToSeconds(permissionToTime + ":00") - DateUtil.
                  convertHhMmSsToSeconds(outPunchDateTime.format(DateTimeFormatter.ofPattern("H:mm:ss")));
              long totalPermissionSeconds = earlyBySeconds - totalPT;
              if (totalPermissionSeconds >= 0) {
                earlyBySeconds = totalPermissionSeconds;
              } else {
                earlyBySeconds = 0;
              }
            } else if (shiftOutGrace.compareTo(permissionFromDateTime) > 0
                && shiftOutGrace.compareTo(permissionToDateTime) <= 0) {
              long totalPT = DateUtil.convertHhMmSsToSeconds(shiftOutGrace.format(DateTimeFormatter.ofPattern("H:mm:ss"))) - DateUtil
                  .convertHhMmSsToSeconds(outPunchDateTime.format(DateTimeFormatter.ofPattern("H:mm:ss")));
              earlyBySeconds = earlyBySeconds - totalPT;
            }
          }
        }
