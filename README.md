private long getFixedPermissionHours(LocalDateTime currentDate,
      long totalFWR, PunchRequest firstPunch, PunchRequest lastPunch){

    SearchRequest searchRequestToGetPermissions = new SearchRequest();
    searchRequestToGetPermissions.getSearchMap().put(SearchKey.ACCOUNT_NO, firstPunch.getEmployeeAccountNo().toString());
    searchRequestToGetPermissions.getSearchMap().put(SearchKey.STATUS, TransactionStatus.APPROVED.toString());
    searchRequestToGetPermissions.getSearchMap().put(SearchKey.START_DATE,
        DateUtil.formatDate(currentDate, DateUtil.DATEFORMAT_yyyyMMdd));
    searchRequestToGetPermissions.getSearchMap()
        .put(SearchKey.AFFILIATE_ID, firstPunch.getAffiliateId().toString());
    searchRequestToGetPermissions.getSearchMap()
        .put(SearchKey.ORGANIZATION_ID, firstPunch.getOrganizationId().toString());

          PermissionDetailsList permissionDetailsList = permissionRestClient.searchPermission(searchRequestToGetPermissions);

          if (permissionDetailsList.getTotalCount() != null
              && permissionDetailsList.getTotalCount() > 0) {
            for (int i = 0; i < permissionDetailsList.getTotalCount(); i++) {
              PermissionDetails permissionDetails = permissionDetailsList.getPermissionDetails().get(i);

              if(permissionDetails.isAddHoursToTheAttendance()){

              LocalTime permissionFromTime = LocalTime
                  .parse(permissionDetails.getFromTime(), DateTimeFormatter.ofPattern("H:mm"));
              LocalDateTime permissionFromDateTime = currentDate.toLocalDate().atTime(permissionFromTime);
              LocalTime permissionToTime = LocalTime
                  .parse(permissionDetails.getToTime(), DateTimeFormatter.ofPattern("H:mm"));
              LocalDateTime permissionToDateTime = currentDate.toLocalDate().atTime(permissionToTime);
              if (firstPunch.getLocalPunchTime().compareTo(permissionFromDateTime) > 0
                  && firstPunch.getLocalPunchTime().compareTo(permissionToDateTime) < 0) {
                long totalPT = DateUtil.convertHhMmSsToSeconds(firstPunch.getLocalPunchTime().
                    format(DateTimeFormatter.ofPattern("H:mm:ss"))) - DateUtil
                    .convertHhMmSsToSeconds(permissionDetails.getFromTime() + ":00");
                totalFWR = totalFWR + totalPT;
              } else if (firstPunch.getLocalPunchTime().compareTo(permissionFromDateTime) > 0) {
                totalFWR = totalFWR + DateUtil.convertHhMmSsToSeconds(permissionDetails.getDifferenceHours());
              }

              if (lastPunch.getLocalPunchTime().compareTo(permissionFromDateTime) > 0
                  && lastPunch.getLocalPunchTime().compareTo(permissionToDateTime) < 0) {
                long totalPT = DateUtil.convertHhMmSsToSeconds(permissionDetails.getToTime() + ":00") -
                    DateUtil.convertHhMmSsToSeconds(lastPunch.getLocalPunchTime().
                    format(DateTimeFormatter.ofPattern("H:mm:ss")));
                totalFWR = totalFWR + totalPT;
              } else if (lastPunch.getLocalPunchTime().compareTo(permissionFromDateTime) <= 0) {
                totalFWR = totalFWR + DateUtil.convertHhMmSsToSeconds(permissionDetails.getDifferenceHours());
              }

             }

            }
          }
    return totalFWR;
  }
