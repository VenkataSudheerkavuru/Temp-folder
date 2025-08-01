/*
        Break Type - Actual
     */
    Map<String, List<PunchRequest>> categorizedPunchesMap_Actual = categorizePunches(punchList,
        BreakType.ACTUAL);
    List<PunchRequest> inPunches_Actual = categorizedPunchesMap_Actual.get("IN");
    List<PunchRequest> outPunches_Actual = categorizedPunchesMap_Actual.get("OUT");

    List<ProcessData> processData = new ArrayList<ProcessData>();

    if (inPunches_Actual.size() > 0 && outPunches_Actual.size() > 0) {
      if (inPunches_Actual.get(0).getLocalPunchTime()
          .isAfter(outPunches_Actual.get(0).getLocalPunchTime())) {
        outPunches_Actual.remove(0);
      }
    }

    if (inPunches_Actual.size() >= 1) {
      determineWorkLocation(workHourDetails, inPunches_Actual.get(0));
    } else if (outPunches_Actual.size() == 1) {
      determineWorkLocation(workHourDetails, outPunches_Actual.get(0));
    }

    // -----------------------------

    AffiliateDetails affiliateDetails = organizationRestClient
        .getAffiliateDetails(workHourPolicy.getAffiliateId());


    if (affiliateDetails.isInOutReport() && (!inPunches_Actual.isEmpty() && !outPunches_Actual.isEmpty())) {

      try {
        LOGGER.info("DELETING RECORD");
        processDataRepository.deleteByEmployeeAccountNoAndProcessDate(inPunches_Actual.get(0).getEmployeeAccountNo(),
            DateUtil.convertLocalDateTimeToDate(currentDate, TimeZone.getDefault()));
        LOGGER.info("DELETING RECORD SUCCESSFUL");
      }
      catch (Exception e){
        throw new BusinessException(WHC01, ExceptionMessages.INTERNAL_SERVER_ERROR);
      }

      for (int i = 0; i <= inPunches_Actual.size() - 1; i++) {
        ProcessData processData1 = new ProcessData();
        // Preparing process_data

        processData1.setEmployeeAccountNo(inPunches_Actual.get(0).getEmployeeAccountNo());
        processData1.setOrganizationId(inPunches_Actual.get(0).getOrganizationId());
        processData1.setAffiliateId(inPunches_Actual.get(0).getAffiliateId());
        processData1.setProcessDate(DateUtil
            .convertLocalDateTimeToDate(currentDate, TimeZone.getDefault()));
        processData1.setEmployeeName(inPunches_Actual.get(0).getEmployeeName());




        for (int j = 0; j <= outPunches_Actual.size() - 1; j++) {
          if (i == j) {
            processData1.setInPunch(inPunches_Actual.get(i).getLocalPunchTime());
            processData1.setOutPunch(outPunches_Actual.get(j).getLocalPunchTime());
            processData1.setInRemark(inPunches_Actual.get(i).getRemarks());
            processData1.setOutRemark(inPunches_Actual.get(j).getRemarks());
            // Calculating work hours between two punches
            if (i == 0) {
              String Wh = calculateWhOrBh(inPunches_Actual.get(i).getLocalPunchTime(),
                  outPunches_Actual.get(j).getLocalPunchTime());
              processData1.setWorkHours(Wh);
                if(inPunches_Actual.get(i).getLocation() != null) {
                 processData1.setWorkLocation(Worklocation(inPunches_Actual.get(i).getLocation().getName()));
                }
            } else {
              String Wh = calculateWhOrBh(inPunches_Actual.get(i).getLocalPunchTime(),
                  outPunches_Actual.get(j).getLocalPunchTime());

              //workHourDetails.setBreakInTime(outPunches_Actual.get(j - 1).getLocalPunchTime().toString());
              //workHourDetails.setBreakOutTime(inPunches_Actual.get(i).getLocalPunchTime().toString());

              workHourDetails.setBreakInTime(DateUtil
                  .formatDate(outPunches_Actual.get(j - 1).getLocalPunchTime(),
                      DateUtil.TIMEFORMAT_Hmm));

              workHourDetails.setBreakOutTime(DateUtil
                  .formatDate(inPunches_Actual.get(i).getLocalPunchTime(),
                      DateUtil.TIMEFORMAT_Hmm));

              String Bh = calculateWhOrBh(outPunches_Actual.get(j - 1).getLocalPunchTime(),
                  inPunches_Actual.get(i).getLocalPunchTime());
              processData1.setWorkHours(Wh);
              processData1.setBreakHours(Bh);
                if(inPunches_Actual.get(i).getLocation() != null) {
                 processData1.setWorkLocation(Worklocation(inPunches_Actual.get(i).getLocation().getName()));
                }
            }
          } else if (i > outPunches_Actual.size() - 1) {
            processData1.setInPunch(inPunches_Actual.get(i).getLocalPunchTime());
            String Bh = calculateWhOrBh(outPunches_Actual.get(j).getLocalPunchTime(),
                inPunches_Actual.get(i).getLocalPunchTime());

            workHourDetails.setBreakInTime(DateUtil
                .formatDate(outPunches_Actual.get(j).getLocalPunchTime(), DateUtil.TIMEFORMAT_Hmm));
            workHourDetails.setBreakOutTime(DateUtil
                .formatDate(inPunches_Actual.get(i).getLocalPunchTime(), DateUtil.TIMEFORMAT_Hmm));

            processData1.setBreakHours(Bh);
            if(inPunches_Actual.get(i).getLocation() != null) {
              processData1.setWorkLocation(Worklocation(inPunches_Actual.get(i).getLocation().getName()));
            }
          }
        }
        processData.add(processData1);
      }
      processData.forEach(processDataRecords1 -> {
        try {
          ProcessDataRecords punchArrangeData = mapper
              .map(processDataRecords1, ProcessDataRecords.class);
          processDataRepository.save(punchArrangeData); // save emp_bg_process_data
        } catch (Exception e) {
          throw new BusinessException(WHC01, ExceptionMessages.INTERNAL_SERVER_ERROR);
        }
      });
    }

    //------------------------------------------------------

    if (inPunches_Actual.size() > 0 && outPunches_Actual.size() > 0) {
      PunchRequest firstPunch = inPunches_Actual.get(0);
      PunchRequest lastPunch = outPunches_Actual.get(outPunches_Actual.size() - 1);
      /*
       * Calculate Actual Work Hours
       **/
      long totalWHSeconds = getActualWorkHours(inPunches_Actual, outPunches_Actual);

      //calculate Permission Hours

      totalWHSeconds = getTotalPermission(inPunches_Actual, outPunches_Actual,
          firstPunch.getEmployeeAccountNo(), currentDate, workHourPolicy, totalWHSeconds);

      workHourDetails.setActualWorkHours(DateUtil.convertSecondsToHhMmSsFormat(totalWHSeconds));

      if (BreakType.ACTUAL.toString()
          .equalsIgnoreCase(workHourPolicy.getNormalShiftBreakType().toString())) {

        /*
       * Calculate Actual break Hours
       **/
        SearchRequest searchRequestToGetPermissions = new SearchRequest();
        searchRequestToGetPermissions.getSearchMap().put(SearchKey.ACCOUNT_NO, firstPunch.getEmployeeAccountNo().toString());
        searchRequestToGetPermissions.getSearchMap().put(SearchKey.STATUS, TransactionStatus.APPROVED.toString());
        searchRequestToGetPermissions.getSearchMap().put(SearchKey.START_DATE,
            DateUtil.formatDate(currentDate, DateUtil.DATEFORMAT_yyyyMMdd));
        searchRequestToGetPermissions.getSearchMap()
            .put(SearchKey.AFFILIATE_ID, workHourPolicy.getAffiliateId().toString());
        searchRequestToGetPermissions.getSearchMap()
            .put(SearchKey.ORGANIZATION_ID, workHourPolicy.getOrganizationId().toString());


        long totalBHSeconds = getActualBreakWorkHours(inPunches_Actual, outPunches_Actual, searchRequestToGetPermissions);
        workHourDetails.setTotalBreakHour(DateUtil.convertSecondsToHhMmSsFormat(totalBHSeconds));

        workHourDetails = calculateInPunchOutPunch(currentDayShift, workHourDetails, firstPunch,
            lastPunch, currentDate,workHourPolicy,shiftProcessingContextModel);
        workHourDetails.setWorkhoursBasedOnBreakType(workHourDetails.getActualWorkHours());
        calculateWHBeforeAfterShift(workHourPolicy, currentDayShift, currentDate, workHourDetails,
            firstPunch, lastPunch);
      }
    } else if (inPunches_Actual.size() >= 1 && BreakType.ACTUAL.toString()
        .equalsIgnoreCase(workHourPolicy.getNormalShiftBreakType().toString())) {
      setInPunchTime(currentDayShift, workHourDetails, inPunches_Actual.get(0), currentDate,workHourPolicy,shiftProcessingContextModel);
    } else if (outPunches_Actual.size() >= 1 && BreakType.ACTUAL.toString()
        .equalsIgnoreCase(workHourPolicy.getNormalShiftBreakType().toString())) {
      setOutPunchTime(currentDayShift, workHourDetails,
          outPunches_Actual.get(outPunches_Actual.size() - 1), currentDate,workHourPolicy,shiftProcessingContextModel);
    }
