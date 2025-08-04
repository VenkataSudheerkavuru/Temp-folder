// Binding data for worked hours
        if (processingRecord.getAttendanceStatusCode().matches("WO|H|LOP|OH") || transactionConfirmation.isLeaveApplied()) {
            if (transactionConfirmation.isLeaveApplied()) {
                if (transactionConfirmation.isIsleaveAppliedForFullDay()) {
                    processingRecord.setWorkedHours("00:00:00");
                } else {
                    if (transactionConfirmation.isSpecialRequestApplied()) {
                        if (transactionConfirmation.isSpecialRequestForFullDay()) {
                            if (currentDayShift.isNormalShiftWithBreakHours()) {
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                            } else if (breakTimeBasedOnPrecedence == null) {
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff)));
                            } else {

                                long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                                fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
                            }
                        }
                        // handling leave and hourly special request combined
                        else if (transactionConfirmation.getIsHourlySpecialRequest()) {
                            //calculate the work hour for hourly special request
                            updateWorkHourForHrlyAndHalfDaySpecialrequest(processingRecord, workHourPolicy, currentDayShift, transactionConfirmation, breakTimeBasedOnPrecedence, specialRequestAbbrCombValidityType, fixedBreaktime, shiftDiff);
                        } else {
                            Long workHr = 0L;
                            if (processingRecord.getWhBasedOnWHPolicy() != null) {
                                workHr = DateUtil.convertHhMmSsToSeconds(processingRecord.getWhBasedOnWHPolicy());
                            }
                            if (currentDayShift.isNormalShiftWithBreakHours()) {
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) - fixedBreakTimeBtwBreakOutIn) / 2) + workHr));
                            } else if (breakTimeBasedOnPrecedence == null) {
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) / 2) + workHr));
                            } else {
                                long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                                if (!CollectionUtils.isEmpty(workHourPolicy.getWorkHourRuleModelList())) {
                                    for (int i = 0; i < workHourPolicy.getWorkHourRuleModelList().size(); i++) {
                                        if (workHourPolicy.getWorkHourRuleModelList().get(i).isEnableBreakType()) {
                                            List<Long> ids = workHourPolicy.getWorkHourRuleModelList().get(i).getIds();
                                            List<Long> longStream = ids.stream().filter(item -> item.equals(processingRecord.getShiftId())).collect(Collectors.toList());
                                            if (longStream != null) {
                                                fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getWorkHourRuleModelList().get(i).getBreakTime());
                                            } else {
                                                fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(breakTimeBasedOnPrecedence);
                                            }
                                        }
                                    }
                                }
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) - fixedBreakTimeInLong) / 2) + workHr));
                            }
                        }
                    } else {
                        // leave applied for half day
                        if (processingRecord.getAttendanceStatusCode().startsWith("A") || processingRecord.getAttendanceStatusCode().endsWith("A")) {
                            processingRecord.setWorkedHours("00:00:00");
                        } else if (processingRecord.getAttendanceStatusCode().startsWith("P") || processingRecord.getAttendanceStatusCode().endsWith("P") || processingRecord.getAttendanceStatusCode().startsWith("M") || processingRecord.getAttendanceStatusCode().endsWith("M") || processingRecord.getAttendanceStatusCode().startsWith("MS") || processingRecord.getAttendanceStatusCode().endsWith("MS")) {
                            processingRecord.setWorkedHours(processingRecord.getWhBasedOnWHPolicy());
                        }
                    }
                }
            } else {
                processingRecord.setWorkedHours("00:00:00");
            }
        } else if (transactionConfirmation.isSpecialRequestApplied()) {

            if (transactionConfirmation.isSpecialRequestForFullDay()) {
                /*processingRecord.setWorkedHours(workHourPolicy.getMinThresholdWorkHourForFullday());*/

                if (specialRequestAbbreviation) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(shiftDiff));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
                    }
                }

            }
            //hourly request
            else if (transactionConfirmation.getIsHourlySpecialRequest()) {

                updateWorkHourForHrlyAndHalfDaySpecialrequest(processingRecord, workHourPolicy, currentDayShift, transactionConfirmation, breakTimeBasedOnPrecedence, specialRequestAbbrCombValidityType, fixedBreaktime, shiftDiff);
            } else {
                /*processingRecord.setWorkedHours(workHourPolicy.getMinThresholdWorkHourForHalfday());*/
                long workHr = 0;
                if (processingRecord.getWhBasedOnWHPolicy() != null) {
                    workHr = DateUtil.convertHhMmSsToSeconds(processingRecord.getWhBasedOnWHPolicy());
                }

                if ((isValidPresentSpecialRequestAbbreviation) || (processingRecord.getAttendanceStatusCode().matches("POD|PFW|PWFH|ODP|FWP|WFHP|ODWFH|WFHOD|ODFW|FWOD|FWWFH|WFHFW|MOD|MFW|MWFH|ODM|FWM|WFHM|MSOD|MSFW|MSWFH|ODMS|FWMS|WFHMS"))) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) - fixedBreakTimeBtwBreakOutIn) / 2) + workHr));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) / 2) + workHr)));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) - fixedBreakTimeInLong) / 2) + workHr));
                    }
                } else if (isValidCombinationSpecialRequestAbbreviation) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(shiftDiff));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        if (!CollectionUtils.isEmpty(workHourPolicy.getWorkHourRuleModelList())) {
                            for (int i = 0; i < workHourPolicy.getWorkHourRuleModelList().size(); i++) {
                                if (workHourPolicy.getWorkHourRuleModelList().get(i).isEnableBreakType()) {
                                    List<Long> ids = workHourPolicy.getWorkHourRuleModelList().get(i).getIds();
                                    List<Long> longStream = ids.stream().filter(item -> item.equals(processingRecord.getShiftId())).collect(Collectors.toList());
                                    if (longStream != null) {
                                        fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getWorkHourRuleModelList().get(i).getBreakTime());
                                    } else {
                                        fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(breakTimeBasedOnPrecedence);
                                    }
                                }
                            }
                        }
                        // long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
                    }
                } else if ((isValidAbsenceSpecialRequestAbbreviation) || (processingRecord.getAttendanceStatusCode().matches("AOD|AFW|AWFH|ODA|FWA|WFHA"))) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeBtwBreakOutIn) / 2));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) / 2)));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeInLong) / 2));
                    }
                } else if ((isValidLOPSpecialRequestAbbreviation) || (processingRecord.getAttendanceStatusCode().matches("LOPFW|FWLOP|ALOP|LOPA|LOPOD|ODLOP|LOPWFH|WFHLOP"))) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeBtwBreakOutIn) / 2));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) / 2)));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeInLong) / 2));
                    }
                }
            }
        } else {
            processingRecord.setWorkedHours(processingRecord.getWhBasedOnWHPolicy());
        }


        // Binding value for Absence Hours
        Long requiredHoursInNumberFormat = DateUtil.convertHhMmSsToSeconds(processingRecord.getRequiredHours());
        Long workedHoursInNumberFormat = DateUtil.convertHhMmSsToSeconds(processingRecord.getWorkedHours());
        Long sum = requiredHoursInNumberFormat - workedHoursInNumberFormat;
        if (sum >= 0) {
            processingRecord.setAbsenceHours(DateUtil.convertSecondsToHhMmSsFormat(sum));
        } else {
            processingRecord.setAbsenceHours("00:00:00");
        }

        if (transactionConfirmation.isAttendanceExceptionEligibility()) {
            processingRecord.setRequiredHours(DateUtil.ZERO_HOUR_HHMMSS);
            processingRecord.setWorkedHours(DateUtil.ZERO_HOUR_HHMMSS);
            processingRecord.setAbsenceHours(DateUtil.ZERO_HOUR_HHMMSS);
        }

    }
